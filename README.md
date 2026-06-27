# Relatório de Análise: Inconsistência no Registro Biométrico de Ponto

## 1. O Problema (Visão de Negócio)
Foi identificado que servidores que foram desligados e, posteriormente, recontratados (ganhando uma nova matrícula) estão tendo problemas na consolidação do ponto. Ao colocarem a digital no leitor biométrico, o sistema reconhece o servidor e **efetua a batida de ponto normalmente, tocando o som de sucesso**, porém **atribui essa marcação à matrícula antiga (inativa)**. Isso faz com que as horas não apareçam na matrícula correta, exigindo ajustes manuais no banco posteriormente.

## 2. A Causa Raiz (Visão Técnica e Banco de Dados)
A origem do problema está na forma como o sistema carrega e identifica as digitais no Backend (aplicação Grails):

1. **Duplicidade Lógica de Cadastro:** Quando um servidor retorna e cadastra a digital na nova matrícula, o banco de dados passa a armazenar **duas digitais** idênticas (ou muito semelhantes) vinculadas a matrículas diferentes para a mesma pessoa.
2. **Carregamento das Digitais (Cache em Memória):** Ao iniciar, a API executa uma consulta (`Digital.findAll()`) que puxa **todas** as digitais do banco de dados para a memória (variável `templatesBD`), sem distinguir se a matrícula atrelada ao servidor está ativa ou possui data de desligamento.
3. **O Matching (Leitor Biométrico):** Quando o usuário coloca o dedo, o SDK biométrico varre esse array em memória e retorna a matrícula vinculada à **primeira digital** que apresentar similaridade suficiente. Se o SDK bater primeiro na digital antiga, o ponto é lançado na matrícula inativa.

> [!WARNING]
> **Por que não dava erro na hora?** A rota via biometria (`registrarPontoBio`) insere a batida de ponto diretamente, sem validar a situação funcional no momento do *insert*. Já a rota "Sem Digital" exibia os erros de "Sem Lotação" que ajudaram a revelar o problema.

## 3. O "Conserto Automático": Por que em alguns dias o erro parava de ocorrer?
Foi relatado que, depois de um tempo, o servidor passava a marcar o ponto na matrícula nova automaticamente, dando a falsa impressão de que o sistema "aprendeu" ou se corrigiu. A análise do código (`PessoaService.groovy`) revela o que realmente acontecia:

1. **A Gaveta de Memória (`templatesBD`):** Quando a API é iniciada, ela vai ao banco de dados, puxa **todas** as digitais (`Digital.findAll()`) e as carrega em uma "gaveta" na memória RAM (um Array). O Leitor Biométrico apenas pesquisa nessa gaveta rápida, e não no banco de dados.
2. **Correção Manual Invisível:** Quando os erros começavam, a equipe de TI precisava migrar/corrigir os dados da matrícula velha para a nova diretamente no banco de dados (via SQL).
3. **O Ponto Cego:** Apesar do banco de dados estar arrumado, a "gaveta" na memória RAM do sistema não ficava sabendo disso, e o sistema continuava registrando na matrícula antiga.
4. **O Efeito Dominó (O Falso Conserto):** A gaveta de memória do sistema só é reconstruída em dois cenários estritos codificados no backend:
   - **Reinicialização:** Quando o servidor reiniciava (numa madrugada ou queda de energia).
   - **Cadastro de Nova Digital:** Quando o RH realizava o cadastro da digital de *qualquer outro* servidor, o código forçava a limpeza e a recarga total de todas as digitais. A análise do código comprovou que a correção "espontânea" e intermitente ocorria porque o cache era integralmente reconstruído no método `salvarDigital()` (linhas 208-209 de `PessoaService.groovy`: `todasDigitais = Digital.findAll(); inicializaTemplateArrays();`).
   
Quando um desses dois eventos ocorria, o sistema puxava as correções manuais que a TI havia feito no banco. A partir desse minuto, o servidor passava a bater o ponto na matrícula nova "como mágica".

## 4. A Solução Proposta (Ação em Runtime)
Para resolver o problema de forma definitiva, sem causar impacto na experiência do usuário e sem exigir recadastramento em massa, a solução foi implementada via código (Runtime) no momento da identificação:

1. **Identificação Inicial:** O SDK continuará varrendo o array e poderá encontrar a matrícula antiga (inativa).
2. **Buscamos qual é a matrícula ATIVA (sem desligamento) desse Título:**
   `SELECT matricula FROM Servidor WHERE titulo = '12345678' AND desligamento IS NULL`
   *Resultado:* `10300997` (A matrícula nova!)

3. **Registramos o Ponto (Desta vez, no lugar certo):**
   Jogamos a matrícula velha fora e mandamos o ponto para a `10300997`.

Dessa forma, pouco importa se a digital que o sistema encontrou foi cadastrada ontem ou há 10 anos atrás. Nós usamos a digital apenas para descobrir **quem é a pessoa** (pelo Título Eleitoral), e então o banco de dados nos diz **qual é o contrato de trabalho atual** dessa pessoa. Ninguém ficará sem conseguir bater ponto!

## 5. Implementação: Código Antes e Depois (Refatorado)

Abaixo estão as correções aplicadas e refinadas para tratar questões de acoplamento, duplicação de regras e edge cases.

### Arquivo: `grails-app/domain/br/jus/treto/ponto/Servidor.groovy`
Centralização da regra de negócio de busca do servidor ativo pelo título, evitando duplicação de código. Adição do alerta de Edge Case.

**Código Antes:**
```groovy
    static constraints = {
        desligamento(blank:true, nullable:true)
    }
```

**Código Depois:**
```groovy
    static constraints = {
        desligamento(blank:true, nullable:true)
    }

    static Servidor buscarServidorAtivoPorTitulo(String tituloBusca) {
        // EDGE CASE CONHECIDO: Se um servidor tiver sido desligado e recontratado mais de uma vez
        // (múltiplas matrículas inativas) e não houver matrícula ativa no momento, o fallback 
        // findByTitulo pode retornar uma matrícula inativa arbitrária entre as inativas.
        return findByTituloAndDesligamentoIsNull(tituloBusca) ?: findByTitulo(tituloBusca)
    }
```
**Observação:**
Extraímos a lógica `findByTituloAndDesligamentoIsNull(login) ?: findByTitulo(login)` para um método estático na classe de domínio e documentamos o limite do fallback. O porquê: se a regra de negócio para definir o que é uma "matrícula ativa" mudar no futuro, alteramos em apenas um lugar, evitando esquecimentos e falhas de sincronia entre os controllers.

---

### Arquivo: `grails-app/services/br/jus/treto/ponto/PessoaService.groovy`
Este é o ponto principal onde a digital é reconhecida. A lógica utiliza o novo método estático para garantir que a matrícula retornada seja sempre a do contrato ativo, e também inclui a documentação de débitos técnicos.

**Código Antes (Método identificarServidor):**
```groovy
    String identificarServidor(byte[] digitalRecebida) {
        int[] templateEncontrado = new int[1]
        int nRes = sdk.UFM_Identify(hMatcher[0], digitalRecebida, digitalRecebida.size(), templatesBD, tamTemplates, templatesBD.length, 20000, templateEncontrado)
        boolean encontrouADigital = nRes == 0 && templateEncontrado[0] != -1
        if (encontrouADigital) {
            return todasDigitais.get(templateEncontrado[0]).matricula
        }
        return null
    }
```

**Código Depois (Método identificarServidor):**
```groovy
    String identificarServidor(byte[] digitalRecebida) {
        int[] templateEncontrado = new int[1]
        int nRes = sdk.UFM_Identify(hMatcher[0], digitalRecebida, digitalRecebida.size(), templatesBD, tamTemplates, templatesBD.length, 20000, templateEncontrado)
        
        if (nRes != 0 || templateEncontrado[0] == -1) {
            return null
        }

        String matriculaEncontrada = todasDigitais.get(templateEncontrado[0]).matricula
        def titulo = Servidor.get(matriculaEncontrada)?.titulo
        
        if (!titulo) {
            return matriculaEncontrada
        }

        // Tenta achar a matrícula ativa usando o método centralizado
        return Servidor.buscarServidorAtivoPorTitulo(titulo)?.matricula ?: matriculaEncontrada
    }
```
**Observação:**
A refatoração melhora a legibilidade (com *early returns*) e faz uso do `.get()`, que é performático, pois a propriedade `matricula` é a chave primária mapeada da classe `Servidor`.

**Código Antes (Método inicializaTemplateArrays):**
```groovy
    def inicializaTemplateArrays() {
        templatesBD = new byte[todasDigitais.size()][1024]
```

**Código Depois (Método inicializaTemplateArrays):**
```groovy
    def inicializaTemplateArrays() {
        // DEBITE TÉCNICO: Há um acoplamento implícito e dependência estrita de índice 
        // entre o array templatesBD e a lista todasDigitais. O índice retornado pelo 
        // SDK no templatesBD é o mesmo usado para buscar a digital em todasDigitais.
        // NOTA SOBRE DUPLICIDADE: Digitais inativas continuam sendo carregadas na memória,
        // gerando peso no SDK a longo prazo.
        templatesBD = new byte[todasDigitais.size()][1024]
```
**Observação:**
Adicionado um registro de **Débito Técnico** e acoplamento. O porquê: o conserto resolve o sintoma mas não limpa as digitais duplicadas do banco, causando um crescimento contínuo do array, o que com o tempo causará lentidão no SDK. A relação de índices, agora explícita via comentário, evita que um dev desavisado altere uma das listas de forma independente e quebre todo o mapeamento no futuro.

---

### Arquivo: `grails-app/controllers/br/jus/treto/ponto/FrequenciaController.groovy`
Corrige o "ponto sem digital", utilizando a nova regra centralizada.

**Código Antes:**
```groovy
        Servidor servidor = Servidor.findByTitulo(login)
```

**Código Depois:**
```groovy
        Servidor servidor = Servidor.buscarServidorAtivoPorTitulo(login)
```
**Observação:**
Centraliza a regra de negócio para buscar a matrícula ativa (caso exista), mantendo o código limpo.

---

### Arquivo: `grails-app/controllers/br/jus/treto/ponto/PessoaController.groovy`
Corrige as permissões e o login da gestão.

**Código Antes:**
```groovy
        Servidor servidor = Servidor.findByTitulo(login)
```

**Código Depois:**
```groovy
        Servidor servidor = Servidor.buscarServidorAtivoPorTitulo(login)
```
**Observação:**
Aplica a mesma lógica centralizada para os fluxos de gestão, removendo a duplicação direta de query e reduzindo o risco de erro humano em manutenções futuras.

## 6. Próximos Passos e Oportunidades de Otimização (Trabalhos Futuros)

A resolução atual atuou como um fix de segurança, mas existem débitos técnicos arquiteturais identificados que devem ser abordados em refatorações futuras para otimizar performance e resiliência:

1. **Custo de Consultas por Leitura Biométrica (I/O):**
   - Atualmente, a identificação biométrica realiza consultas ao banco (`Servidor.buscarServidorAtivoPorTitulo`) para checar o status da matrícula a cada leitura. 
   - **Solução proposta:** Como o sistema já possui gatilhos bem definidos de recarga de cache (reinício e `salvarDigital`), recomenda-se futuramente carregar um **mapa em memória (`título -> matrícula ativa`)** nestes mesmos momentos, eliminando o custo de consulta ao banco a cada batida de ponto.

2. **Carga Desnecessária de Digitais Inativas (Uso de Memória):**
   - O array do SDK continua carregando as biometrias de todas as matrículas, incluindo as inativas. Isso é um acoplamento implícito e gera peso desnecessário no algoritmo do SDK a longo prazo.
   - **Solução proposta:** Modificar a inicialização do cache (`Digital.findAll()`) para filtrar e carregar **apenas** digitais vinculadas a matrículas ativas (ou manter apenas a digital mais recente por título). Isso reduzirá drasticamente o array do SDK, melhorando o tempo de resposta e o uso de memória do servidor.
   
3. **Race Condition na Recarga do Cache de Digitais (Concorrência):**
   - **O Risco:** Durante o recadastro biométrico (`salvarDigital`), o cache é atualizado em dois passos sequenciais: a substituição da lista `todasDigitais` e a posterior recriação do array `templatesBD`. Como o serviço atua em padrão *Singleton*, se uma leitura ocorrer nesse intervalo (fração de segundo), o sistema fará a busca biométrica (`identificarServidor`) usando o array antigo, retornando um índice que pode apontar para um servidor incorreto na nova lista, gerando um registro indevido de ponto para outra pessoa.
   - **Solução proposta:** Otimizar a recarga do cache construindo o novo array e a nova lista em escopo local durante o `salvarDigital`, e realizar a substituição (atribuição aos campos da classe) de forma sincronizada (`synchronized`) ou atômica. Isso garante que a leitura biométrica sempre consuma o par de lista e array de forma íntegra.
