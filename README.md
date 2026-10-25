# Osprey

Revisão sem contexto dos testes contra o plano, para Claude Code.

O [Elephant-Goldfish Model](https://github.com/google/elephant-goldfish-model) revisa o documento. O Osprey revisa os testes que vão cobrar o documento: três revisores isolados atacam os testes de cada unidade do plano, um refutador derruba o que não se sustenta, o que é técnico é corrigido sozinho e só as decisões de negócio chegam até você, como perguntas A/B.

## Instalação

```sh
claude plugin marketplace add blemz/osprey
claude plugin install osprey@osprey
```

Atualizar depois: `claude plugin marketplace update osprey` e `claude plugin update osprey@osprey`.

## Uso

```
/osprey:review docs/plano.md tests/
```

- Primeiro argumento: o plano ou a spec. Segundo: a pasta ou os arquivos de teste.
- Opcional, um terceiro: a unidade (`Task 4`) para revisar só ela.
- Relatório em `docs/osprey/<plano>/REVISAO.md`. Para mudar a pasta, ponha no prompt ou no `CLAUDE.md`: `Osprey directory: <caminho>`.
- Regras próprias do repositório para o auditor: `Osprey rules: <caminho>` no prompt ou no `CLAUDE.md`. Sem isso, valem as regras padrão de `agents/auditor.md`.

## Como funciona

1. **Unidades.** Cada Task do plano é uma unidade; sem Tasks, o plano inteiro é uma.
2. **Três revisores por unidade, em paralelo, sem contexto** (`omitClaudeMd`, só leitura):
   - `osprey:leitor` diz, só pelos asserts, o que cada teste prova e onde diverge do plano;
   - `osprey:sabotador` procura a implementação trapaceira que passaria em todos os testes;
   - `osprey:auditor` confere as regras.
3. **Confirmação.** Achado de dois revisores conta; achado isolado passa por um `osprey:refutador` (um por unidade).
4. **Triagem.** Só se corrige sozinho o que tem âncora literal no plano ou nas regras e endurece o teste. O que afrouxaria um teste, mudaria contrato, dado, segurança ou regra de negócio vira pergunta.
5. **Você responde** às perguntas A/B, em linguagem de produto, com uma recomendação em cada.

## Fluxo com o EGM

```
/egm:egm-review <plano> implementation   # revisa o plano
(escrever os testes)
/osprey:review <plano> <testes>          # revisa os testes contra o plano
(implementar)
```

## Custo

Cerca de 135 mil tokens por unidade nos três revisores, mais o refutador. Num plano de 15 Tasks, algo entre 2,5 e 3 milhões de tokens. Para economizar, revise só as unidades com rota, dado persistido ou regra de negócio.
