---
name: review
description: Use when tests written for a plan or spec must be stress-tested by zero-context reviewers before implementation or approval, so the human only answers business questions instead of reading test code. Invoked as /osprey:review <plano> <testes>.
argument-hint: <plano.md> <pasta-ou-arquivos-de-teste> [unidade]
---

# Osprey: revisão dos testes contra o plano

## Entrada e saída

- `$ARGUMENTS`: o caminho do plano (ou da spec) e o caminho dos testes que o implementam (pasta, arquivo ou lista). Opcionalmente, uma unidade só (por exemplo `Task 4`) para revisar só ela. Se faltar o plano ou os testes, pergunte em texto e pare.
- Regras do auditor: se o prompt ou o `CLAUDE.md`/`AGENTS.md` do repositório tiver a linha `Osprey rules: <caminho>`, o auditor usa esse arquivo; senão, usa as regras padrão que ele já traz.
- Saída: o relatório vai para `<pasta>/<nome-do-plano>/REVISAO.md`. A pasta é a da linha `Osprey directory: <caminho>` no prompt ou no `CLAUDE.md`/`AGENTS.md`; sem ela, `docs/osprey/`. Correções técnicas vão direto nos arquivos de teste.
- A skill nunca edita o plano, nem código de produção, nem commita.

## Por que existe

Quem escreveu os testes não deve julgá-los, e o humano não deve ter de ler `assert` para confiar neles. O Osprey põe três revisores sem contexto contra os testes, confirma o que eles acharem, corrige sozinho o que é técnico e entrega ao humano só as decisões que mudam o que o sistema faz, em linguagem de produto. Ele complementa o EGM: o EGM revisa o documento; o Osprey revisa os testes que vão cobrar o documento.

## Passo 1: unidades

Leia o plano e divida-o em unidades de revisão:

- se o plano tem Tasks (`### Task N`, `## Task N` ou equivalente), cada Task é uma unidade; associe a cada uma os arquivos de teste que ela cria ou que a lista de arquivos dela cita (ou a pasta `task_NN`, se existir);
- se não tem, o plano inteiro é uma unidade e todos os testes vão com ela.

Teste que não casa com nenhuma unidade entra na unidade mais próxima pelo assunto; diga isso no relatório. Unidade sem nenhum teste vira um achado direto: "comportamento do plano sem teste". Se a revisão for de uma unidade só, siga só com ela.

## Passo 2: disparo dos revisores

Para cada unidade, dispare em paralelo, num único lote de chamadas, os três revisores, cada um num subagente novo:

- `osprey:leitor`: diz, só pelos asserts, o que cada teste prova e onde isso diverge do plano;
- `osprey:sabotador`: procura a implementação preguiçosa ou trapaceira que passaria nos testes sem cumprir o plano;
- `osprey:auditor`: confere os testes contra as regras (as do repositório, se configuradas, ou as padrão).

O prompt de cada um traz **só caminhos absolutos**, nunca opinião nem resumo desta conversa:

```text
Raiz do repositório: <caminho absoluto>
Plano: <caminho do plano>
Unidade: <nome da unidade, ex.: Task 4> (leia a seção dela e as restrições globais do plano; das unidades de que ela depende, só as interfaces)
Testes: <arquivos ou pasta>
Apoio: <fixtures, conftest, fábricas ou helpers que os testes usam>
Regras: <caminho do arquivo de regras, ou "padrão">
```

Se o subagente nomeado não existir na sessão, dispare um subagente de uso geral e mande-o ler primeiro o arquivo do agente em `${CLAUDE_PLUGIN_ROOT}/agents/<nome>.md`. Se não houver ferramenta de subagente, pare: revisão no mesmo contexto não é revisão sem contexto.

Resposta que não for JSON válido conta como zero achados daquele revisor e entra nos indicadores.

## Passo 3: confirmação

Agrupe os achados que falam do mesmo problema no mesmo teste (ou na mesma lacuna da unidade).

- Apontado por 2 ou mais revisores → **confirmado**.
- Apontado por um só → por unidade, dispare **um** `osprey:refutador` novo, com a linha `Tipo: testes`, os mesmos caminhos do Passo 2 e a lista numerada dos achados isolados daquela unidade (texto literal de cada revisor). `confirmado` fica; `refutado` vai para descartados com o motivo dele.

Nenhum achado segue sem passar por um dos dois caminhos.

## Passo 4: triagem

Classifique cada achado confirmado. A pergunta que decide é: **a correção muda o que o sistema faz, ou só como o teste prova o que o plano já diz?**

Antes, duas travas. Elas valem acima dos exemplos abaixo:

- **Âncora no plano.** Só se corrige sozinho achado cuja `evidencia_plano` é trecho literal do plano ou regra literal do arquivo de regras (ou das regras padrão do auditor), e que diz o que o achado afirma. Confira você mesmo no arquivo. Sem essa âncora, o achado vira pergunta se muda comportamento; senão, é descartado com o motivo "sem âncora no plano".
- **Só endurecer.** Achado cuja correção removeria um teste, um assert ou um caso, ou aceitaria um resultado que o teste hoje recusa, nunca é corrigido sozinho. Ele vira pergunta: o humano decide se aquilo faz parte do contrato.

Depois:

- **Corrigir sozinho**: passa nas duas travas e a correção não muda o comportamento descrito no plano. Exemplos: assert fraco (só status, `is not None`, "não levantou"), persistência conferida pelo objeto recém-gravado e não por leitura independente, um caso só ou dado fixo que dá para decorar, teste que passa antes de o código existir, erro do contrato que o plano já exige e ninguém testa.
- **Perguntar ao humano**: qualquer escolha pode estar errada e muda contrato (rota, status, formato), dado persistido, segurança, privacidade ou regra de negócio; ou o plano admite duas leituras razoáveis e o teste escolheu uma.
- **Descartar**: estilo, nome, preferência sem efeito no que se prova.

Na dúvida entre corrigir e perguntar, pergunte. Mais de 5 perguntas num plano é sinal de plano ambíguo: diga isso no topo do relatório.

## Passo 5: correção

Aplique as correções técnicas nos arquivos de teste, seguindo o estilo e as fixtures que eles já usam. Depois, confira que os testes ainda são válidos sem depender da implementação: se o projeto tem um comando que só coleta ou compila os testes (por exemplo `pytest --collect-only -q`, `tsc --noEmit`, `go vet ./...`), rode-o; se os testes rodam e o código ainda não existe, eles têm de falhar.

Para cada unidade que teve teste corrigido, dispare **uma vez** um `osprey:leitor` novo só sobre os testes alterados. Divergência que ele ainda apontar vira pergunta ao humano, nunca nova rodada de correção.

## Passo 6: perguntas ao humano

Cada pergunta é escrita para quem conhece o negócio e não lê código:

- uma frase de contexto sobre a situação do usuário (nada de nome de arquivo, função ou `assert`);
- a pergunta;
- opção A (a recomendada, marcada como tal) e opção B, cada uma com o que o usuário veria;
- uma linha de "por que recomendo A".

Se a ferramenta AskUserQuestion estiver disponível, faça as perguntas por ela (até 4 por chamada) e aplique as respostas nos testes. Senão, grave-as no relatório e espere a resposta no chat. Se a resposta mostrar que o plano está errado ou vago, diga ao humano qual trecho do plano corrigir; o plano é dele.

## REVISAO.md

```markdown
# Revisão Osprey: <nome do plano>

Plano: <caminho> · Testes: <caminhos> · Regras: <arquivo ou padrão>

## Decisões para você

### D1. <assunto em poucas palavras>
<contexto em uma frase, sem código>
**Pergunta:** <pergunta>
- **A (recomendada):** <o que o usuário veria>
- **B:** <o que o usuário veria>
Por que A: <uma linha>
**Sua resposta:** <pendente | A | B | outra>
Referência técnica: <arquivo::teste>

## Corrigido sozinho

- <unidade>: <uma linha, em linguagem simples, do que ficou mais forte>

## Descartado

- <n> achados de estilo; <n> refutados (motivo em uma linha cada)

## Indicadores

| Achados brutos | Confirmados | Corrigidos | Perguntados | Descartados | Revisores sem resposta válida |
| --- | --- | --- | --- | --- | --- |
```

No fim, diga ao humano em duas linhas quantas decisões ficaram (e quais estão pendentes) e onde está o relatório.

## Nunca

- Editar o plano, código de produção, configuração de teste (pytest.ini, jest.config etc.) ou commitar.
- Passar aos revisores esta conversa, resumos seus, relatórios do EGM ou a opinião de outro revisor (salvo o achado literal para o refutador).
- Revisar no mesmo contexto quando não houver subagente.
- Fazer pergunta ao humano em termos de código, ou perguntar algo técnico que o Passo 4 manda corrigir sozinho.
- Afrouxar um teste para resolver um achado: correção técnica só endurece.
