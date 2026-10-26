---
name: triage
description: Use when an EGM review (or any reviewer report) produced findings on a design doc, spec or plan, and they must be refuted, fixed when technical and asked as business questions otherwise, instead of the human approving each finding. Invoked as /osprey:triage <relatório> [documento].
argument-hint: <relatório-do-egm.md> [documento-revisado.md]
---

# Osprey: triagem dos achados do EGM

## Entrada e saída

- `$ARGUMENTS`: o caminho de um relatório de revisão do EGM (`critic_report.md`, `implementation_readiness_report.md`, `comprehension_report.md`, `consolidated_review_report.md`, `goldfish_review_report.md` ou outro com a mesma forma: achados com revisor, gravidade, trecho e descrição) e, opcionalmente, o documento revisado. Sem o documento, use o `Target:` do relatório; se não houver, pergunte em texto e pare.
- Saída: o documento editado só nas correções técnicas e o relatório `<pasta>/<nome-do-documento>/TRIAGEM.md`. A pasta é a da linha `Osprey directory: <caminho>` no prompt ou no `CLAUDE.md`/`AGENTS.md`; sem ela, `docs/osprey/`.
- A skill não edita o relatório do EGM, não toca em código e não commita.

## Quando rodar

Depois que o EGM escreveu o relatório e **antes** de resolver os achados nele. No gate dos Stages 7 ou 9 do `/egm:egm-review`, escolha pausar (ou editar o documento manualmente), rode esta skill e, quando ela terminar, avise o EGM que as edições estão feitas, para ele re-auditar (o EGM permite uma re-auditoria por estágio). Esta skill substitui o `egm-resolve` e a aprovação achado por achado.

## Passo 1: leitura

1. Leia o relatório e numere cada achado: revisor, gravidade, trecho do documento (linhas ou seção), descrição e justificativa, copiados literalmente.
2. Leia o documento inteiro e as fontes que ele cita (spec, outros documentos, arquivos de código). Anote as seções em que o documento registra decisões tomadas, fora de escopo ou não objetivos: achados que contestam esses trechos nunca são corrigidos sozinhos.
3. Agrupe achados que falam do mesmo problema no mesmo trecho. Achado que o relatório atribui a 2 ou mais revisores já conta como apontado por dois.

## Passo 2: confirmação

- Achado de 2 ou mais revisores → **confirmado**.
- Os demais: por seção do documento (até 10 achados por chamada), dispare **um** `osprey:refutador` novo com este prompt, só caminhos e o texto literal dos achados:

```text
Tipo: documento
Raiz do repositório: <caminho absoluto>
Documento: <caminho do documento revisado>
Trecho: <seção ou linhas>
Fontes: <spec e arquivos que o documento cita>
Achados:
1. <achado literal>
2. <achado literal>
```

`confirmado` segue; `refutado` vai para descartados com o motivo dele. Se não houver ferramenta de subagente, pare: refutar no mesmo contexto que vai corrigir não é revisão sem contexto.

## Passo 3: triagem

A pergunta que decide é: **a correção só faz o documento dizer, com clareza, o que uma fonte já diz, ou ela escolhe algo?**

Duas travas, acima dos exemplos:

- **Âncora literal.** Só se corrige sozinho achado cuja correção vem de uma fonte que você consegue citar literalmente: outro trecho do mesmo documento, a spec, um documento citado ou o código existente. Sem citação, não há correção automática.
- **Não tirar nada.** Correção que remove ou enfraquece requisito, restrição, salvaguarda, item de escopo ou critério de aceite, ou que muda uma decisão registrada no documento, nunca é feita sozinha.

Depois:

- **Corrigir sozinho**: passa nas duas travas. Exemplos: termo usado sem definição que o próprio documento ou a spec define; contradição entre dois trechos em que um deles é a decisão registrada; interface, nome, caminho ou assinatura que o código existente já fixa e o documento escreveu errado ou deixou vago; passo implícito que o resto do documento pressupõe; referência quebrada; ambiguidade de redação com uma única leitura compatível com as fontes.
- **Perguntar ao humano**: a correção escolhe entre caminhos razoáveis, ou mexe em escopo, contrato (rota, formato, status), dado persistido, segurança, privacidade, custo, prazo ou regra de negócio; contradição sem fonte que desempate; achado que contesta uma decisão registrada.
- **Descartar**: refutado; estilo que contraria a convenção do próprio documento; achado sobre algo declarado fora de escopo.

Na dúvida entre corrigir e perguntar, pergunte. Mais de 5 perguntas é sinal de documento ainda imaturo: diga isso no topo da `TRIAGEM.md` e sugira voltar ao brainstorm ou ao draft.

## Passo 4: correção

Aplique as correções no documento com edições mínimas, no tom e no formato dele, uma por achado. Para cada uma, guarde para o relatório: o trecho antes, o trecho depois (até 3 linhas cada) e a fonte citada. Não reescreva seções inteiras nem melhore o que nenhum achado apontou.

## Passo 5: perguntas ao humano

Cada pergunta é escrita para quem conhece o negócio e não lê código:

- uma frase de contexto sobre a situação (sem jargão, sem nome de função);
- a pergunta;
- opção A (a recomendada, marcada como tal) e opção B, cada uma com o efeito prático;
- uma linha de "por que recomendo A".

Se a ferramenta AskUserQuestion estiver disponível, faça as perguntas por ela (até 4 por chamada) e aplique as respostas no documento. Senão, grave-as na `TRIAGEM.md` e espere a resposta no chat. Cada resposta aplicada entra no documento como decisão registrada, na seção de decisões dele se houver.

## TRIAGEM.md

```markdown
# Triagem Osprey: <nome do documento>

Documento: <caminho> · Relatório do EGM: <caminho>

## Decisões para você

### D1. <assunto em poucas palavras>
<contexto em uma frase>
**Pergunta:** <pergunta>
- **A (recomendada):** <efeito>
- **B:** <efeito>
Por que A: <uma linha>
**Sua resposta:** <pendente | A | B | outra>
Achados do EGM: <números>

## Corrigido sozinho

- Achado <n> (<revisor>): <uma linha do que mudou> · fonte: <citação curta>

## Descartado

- Achado <n>: <refutado | fora de escopo | estilo> — <motivo em uma linha>

## Indicadores

| Achados do EGM | Confirmados | Corrigidos | Perguntados | Descartados |
| --- | --- | --- | --- | --- |
```

No fim, diga ao humano em duas linhas quantas decisões ficaram e que o próximo passo é avisar o EGM para re-auditar o estágio.

## Nunca

- Editar o relatório do EGM, código, testes ou commitar.
- Corrigir sozinho sem fonte citável, ou removendo e enfraquecendo requisito, restrição, salvaguarda, escopo ou decisão registrada.
- Passar ao refutador esta conversa, resumos seus ou a correção que você pretende fazer.
- Fazer pergunta ao humano em termos de código, ou perguntar algo que o Passo 3 manda corrigir sozinho.
