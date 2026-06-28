# Solução A — Endpoint de Refresh do Cache de Digitais (Recarga sob Demanda)

## 1. Objetivo

Permitir que a equipe de banco **force a recarga do cache de digitais em memória** logo após uma migração ou correção via SQL, **sem reiniciar a aplicação** e sem depender do recadastro de digitais por parte do RH.

Hoje o sistema só reconstrói o cache (`todasDigitais` + `templatesBD`) em dois momentos:

1. **Reinício da aplicação** (madrugada, queda de energia, deploy).
2. **Cadastro de uma digital qualquer** (`PessoaService.salvarDigital()`), que dispara `Digital.findAll()` + `inicializaTemplateArrays()` como efeito colateral.

> Referência: ver seções 3 e 7 de [`analise_problema_ponto.md`](./analise_problema_ponto.md).

Esta solução adiciona um **terceiro gatilho explícito e controlado**: um endpoint HTTP de recarga.

---

## 2. Por que "já está praticamente pronto"?

A lógica de recarga **já existe e já está isolada** no código. O método `salvarDigital()` em `PessoaService.groovy` (linhas 208-209) faz exatamente isto:

```groovy
todasDigitais = Digital.findAll()
inicializaTemplateArrays()
```

Ou seja, **não precisamos escrever a lógica de recarga do zero** — ela já está validada e em produção (é o mesmo caminho do "falso conserto" descrito no documento de análise). Precisamos apenas:

1. **Extrair** essas duas linhas para um método público dedicado.
2. **Sincronizar** a troca de referências (resolvendo de quebra a *race condition* da seção 7.3 da análise).
3. **Expor** esse método através de um endpoint HTTP protegido.

É uma mudança pequena, de baixo risco, reaproveitando código já existente.

---

## 3. Fluxo proposto (visão do DBA)

```
┌────────────────────────────┐
│  1. DBA roda script SQL    │
│     UPDATE SRH2.SERVIDOR   │
│     SET DT_DESLIG = ...    │
│     COMMIT;                │
└────────────┬───────────────┘
             │
┌────────────▼──────────────┐
│  2. DBA chama o endpoint  │
│     POST /monitoramento/  │
│         recarregarCache   │
└────────────┬──────────────┘
             │
┌────────────▼───────────────────────────┐
│  3. App executa (sincronizado):        │
│     todasDigitais = Digital.findAll()  │
│     inicializaTemplateArrays()         │
└────────────┬───────────────────────────┘
             │
┌────────────▼─────────────┐
│  4. Cache agora reflete  │
│     o banco — sem        │
│     reiniciar nada       │
└──────────────────────────┘
```

A partir do passo 4, qualquer batida de ponto via biometria já enxerga os dados migrados.

---

## 4. Implementação

### 4.1. `PessoaService.groovy` — novo método sincronizado

Extrair a recarga para um método público e protegê-la com `synchronized`, garantindo que nenhuma leitura biométrica (`identificarServidor`) consuma a lista e o array em estado inconsistente.

```groovy
/**
 * Recarrega o cache de digitais a partir do banco de dados.
 *
 * Monta a nova lista e o novo array em variaveis LOCAIS e so entao troca
 * as referencias de classe dentro de um bloco sincronizado. Isso evita a
 * race condition em que uma leitura biometrica simultanea usaria o array
 * antigo com um indice que aponta para a lista nova (ver secao 7.3 da analise).
 *
 * @return int quantidade de digitais carregadas
 */
int recarregarCache() {
    List<Digital> novasDigitais = Digital.findAll()

    // Monta os arrays do SDK em escopo local (operacao mais cara, fora do lock)
    int qtd = novasDigitais.size()
    byte[][] novosTemplates = new byte[qtd][1024]
    int[] novosTamanhos = new int[qtd]
    for (int i = 0; i < qtd; i++) {
        def tam = (int) novasDigitais[i].getImagemDigital().length()
        novosTemplates[i] = novasDigitais[i].getImagemDigital().getBytes(1, tam)
        novasDigitais[i].getImagemDigital().free()
        novosTamanhos[i] = tam
    }

    // Troca atomica das referencias (rapida, dentro do lock)
    synchronized (this) {
        this.todasDigitais = novasDigitais
        this.templatesBD = novosTemplates
        this.tamTemplates = novosTamanhos
    }

    log.info "[recarregarCache] Cache recarregado. Total de digitais: ${qtd}"
    return qtd
}
```

> **Observação sobre `salvarDigital()` e `identificarServidor()`:** para que a sincronização seja efetiva, idealmente esses métodos também devem ler/escrever o trio (`todasDigitais`, `templatesBD`, `tamTemplates`) de forma coerente com o mesmo `synchronized (this)`. No mínimo, `salvarDigital()` deveria passar a chamar `recarregarCache()` em vez de duplicar as duas linhas, eliminando a duplicação.

### 4.2. `MonitoramentoController.groovy` — endpoint protegido

Adicionar o endpoint no controller que já concentra ações operacionais. **Importante: protegê-lo** (ver seção 5).

```groovy
PessoaService pessoaService
UtilService utilService

/**
 * Recarrega o cache de digitais a partir do banco.
 * Usado pela equipe de banco apos migracoes/correcoes via SQL.
 *
 * Apenas POST e apenas de IPs autorizados da rede interna.
 */
def recarregarCache() {
    // 1. So aceita POST (acao com efeito colateral)
    if (request.method != 'POST') {
        response.status = 405 // Method Not Allowed
        render new Retorno(sucesso: false, informacao: 'ERR_METODO') as JSON
        return
    }

    // 2. Restringe por IP (reusa a regra ja existente no projeto)
    def ip = request.remoteAddr
    if (utilService.isProducao() && !utilService.ipPertenceTRE(ip)) {
        log.warn "ERR02-[recarregarCache] Tentativa de recarga fora da rede do TRE. IP: ${ip}"
        render new Retorno(sucesso: false, informacao: 'ERR02') as JSON
        return
    }

    try {
        int qtd = pessoaService.recarregarCache()
        log.warn "[recarregarCache] Cache recarregado sob demanda pelo IP ${ip}. Total: ${qtd}"
        render new Retorno(sucesso: true, informacao: "Cache recarregado. ${qtd} digitais.") as JSON
    } catch (Exception e) {
        log.error "ERR05-[recarregarCache] Falha ao recarregar cache: ${e.message}"
        render new Retorno(sucesso: false, informacao: 'ERR05') as JSON
    }
}
```

O roteamento já é coberto pelo `UrlMappings.groovy` padrão (`/$controller/$action?/$id?`), então a URL fica automaticamente disponível como:

```
POST /monitoramento/recarregarCache
```

Não é preciso alterar o `UrlMappings`.

---

## 5. Segurança (obrigatório)

Um endpoint de recarga **não pode ficar aberto**. Recarregar o cache é uma operação cara (lê todas as digitais do banco e remonta os arrays do SDK); chamadas repetidas seriam um vetor de **negação de serviço (DoS)**.

Controles aplicados na proposta acima:

| Controle | Como | Por quê |
|---|---|---|
| **Somente POST** | Checagem de `request.method` | Ação com efeito colateral não deve ser GET (evita disparo acidental por crawler/navegador) |
| **Restrição por IP** | `utilService.ipPertenceTRE(ip)` (já existe no projeto) | Garante que só a rede interna/banco chama |
| **Log de auditoria** | `log.warn` com IP e quantidade | Rastreabilidade de quem recarregou e quando |

**Reforço recomendado** (opcional, conforme política do órgão): exigir um token/segredo no header (ex.: `X-Refresh-Token`) comparado com um valor em `application.yml`, para o caso de a restrição de IP não ser suficiente.

---

## 6. Como usar (exemplo)

Depois de rodar os scripts de migração e dar `COMMIT`:

```bash
# A partir de uma máquina dentro da rede autorizada:
curl -X POST http://<host-ponto>:<porta>/monitoramento/recarregarCache
```

Resposta esperada:

```json
{ "sucesso": true, "informacao": "Cache recarregado. 1532 digitais." }
```

A partir desse instante, as batidas de ponto já usam os dados migrados — **sem reiniciar a aplicação**.

---

## 7. Benefícios e limitações

**Benefícios:**
- Elimina a dependência de reinício ou de recadastro de digital para aplicar correções de banco.
- Reaproveita lógica já existente e validada (baixo risco).
- Resolve, de quebra, a *race condition* documentada na seção 7.3 da análise (a troca de referências passa a ser sincronizada).
- Ferramenta controlada e auditável para o fluxo de migração.

**Limitações (continuam valendo as observações da seção 7 da análise):**
- A recarga continua carregando **digitais inativas** na memória (não resolve o crescimento do array do SDK — isso é o item 7.2 da análise, "carregar apenas digitais ativas").
- É uma recarga **total** (`findAll`), não incremental. Para a escala atual é aceitável; em bases muito grandes, avaliar recarga parcial no futuro.
- Não substitui a correção de runtime já implementada (`buscarServidorAtivoPorTitulo`); é **complementar** a ela.

---

## 8. Relação com a solução já implementada

| Mecanismo | O que resolve | Estado |
|---|---|---|
| `buscarServidorAtivoPorTitulo` (runtime) | Garante que a **batida de ponto** vá para a matrícula ativa, mesmo com cache desatualizado | ✅ Já implementado |
| **Endpoint de refresh (esta proposta)** | Permite **sincronizar o cache com o banco** sob demanda após migrações | 🔧 A implementar (praticamente pronto) |
| Cache só de digitais ativas (seção 7.2) | Reduz peso no SDK | 🔮 Trabalho futuro |

> Mesmo com a correção de runtime já ativa, o endpoint de refresh continua útil: ele mantém a memória coerente com o banco para **todos** os fluxos que ainda dependem do cache do SDK (o *matching* da digital em si), não só da resolução da matrícula.
