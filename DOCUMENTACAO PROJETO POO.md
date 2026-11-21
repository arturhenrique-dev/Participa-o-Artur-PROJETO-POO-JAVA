# 📘 Documentação — Minha parte no Sistema de Planejamento Financeiro

**Autor:** Artur Henrique · **Curso:** ADS (Unifip) · **Linguagem:** Java · **Projeto:** Integrador de POO

Este documento explica, de forma prática e sem pressupor conhecimento prévio, tudo o que eu desenvolvi no projeto. Pode ser lido por quem está começando e também serve de apoio para a apresentação.

---

## 1. O projeto em uma frase

> É um **caderninho de finanças digital**: a pessoa anota o que ganha e o que gasta, define um **teto de gastos**, e o sistema impede que ela passe desse teto e mostra relatórios de como está o dinheiro.

Cada anotação no caderninho se chama **lançamento**. Pode ser uma **receita** (dinheiro que entra, como o salário) ou uma **despesa** (dinheiro que sai, como o aluguel).

## 2. Minha parte: 3 peças

O projeto é feito por três pessoas. Eu fiquei com o **núcleo financeiro**, que tem três peças:

| Peça | Analogia do dia a dia | O que faz |
|---|---|---|
| `StatusLancamento` | **Etiqueta** colada em cada anotação | Diz em que situação está o lançamento: pendente, confirmado ou cancelado |
| `ContaPlanejamento` | O **caderno e o seu guardião** | Guarda todos os lançamentos e não deixa ninguém quebrar as regras |
| `RelatorioFinanceiro` | O **resumo** que você mostraria ao contador | Lê o caderno e mostra os números organizados |

```
        ┌──────────────────────┐
        │   ContaPlanejamento  │  ← guarda os lançamentos e aplica as regras
        └───────────┬──────────┘
                    │ é lida por
                    ▼
        ┌──────────────────────┐
        │  RelatorioFinanceiro │  ← só lê e mostra, nunca altera
        └──────────────────────┘

  StatusLancamento = as 3 "etiquetas" que cada lançamento carrega
```

---

## 3. `StatusLancamento` — a etiqueta

### O que é
Um **enum**: uma lista fixa de opções permitidas. Em vez de escrever textos soltos como `"pendente"` (onde um erro de digitação passaria despercebido), o Java só aceita um dos três valores abaixo.

### Os três estados

| Estado | Significado | Afeta o saldo atual? |
|---|---|---|
| `PENDENTE` | Anotei, mas ainda não aconteceu de verdade | Não |
| `CONFIRMADO` | Aconteceu (o dinheiro entrou ou saiu) | **Sim** |
| `CANCELADO` | Desisti; ficou só no histórico | Não |

### Como o estado muda

```
PENDENTE ──confirmar──▶ CONFIRMADO ──cancelar──▶ CANCELADO
    │                                                 ▲
    └─────────────────────cancelar────────────────────┘

CANCELADO é um beco sem saída: não volta para nenhum outro estado.
```

### Exemplo de uso
```java
StatusLancamento s = StatusLancamento.CONFIRMADO;
System.out.println(s.getDescricao());   // Confirmado
```

---

## 4. `ContaPlanejamento` — o caderno e o guardião

### O que é
É a conta do usuário. Ela guarda:

- o **limite de orçamento** (o teto de gastos, por exemplo R$ 4.000);
- o **saldo inicial** (quanto a pessoa já tinha ao abrir a conta);
- a **lista de lançamentos**.

Seu papel mais importante é ser o **guardião**: tudo passa por ela, então as regras não podem ser burladas.

### Duas estruturas de dados, cada uma com um motivo

| Estrutura | Para que serve | Analogia |
|---|---|---|
| `ArrayList` | Guarda os lançamentos **na ordem em que foram feitos** e permite percorrer todos | Uma fila de papéis na gaveta |
| `HashMap` | Encontra um lançamento **direto pelo código**, sem folhear tudo | O índice no fim de um livro |

Sem o `HashMap`, para achar o lançamento 57 eu teria que olhar um por um. Com ele, vou direto.

### Operações (os "botões" da conta)

| Método | O que faz |
|---|---|
| `registrarLancamento(l)` | Adiciona um novo lançamento (com todas as validações) |
| `buscarPorCodigo(codigo)` | Acha um lançamento pelo código |
| `atualizarLancamento(codigo, nome, descricao, valor)` | Altera nome, descrição e valor |
| `cancelarLancamento(codigo)` | Muda o status para `CANCELADO` |
| `excluirLancamento(codigo)` | Remove do caderno (só em casos permitidos) |
| `listarLancamentos()` | Devolve uma **cópia somente de leitura** da lista |
| `calcularTotalDespesas()` | Soma das despesas ativas (pendentes + confirmadas, com taxa) |
| `calcularSaldoAtual()` | Saldo inicial + efeito dos lançamentos **confirmados** |

> **Por que `listarLancamentos()` devolve uma cópia?** Se devolvesse a lista original, qualquer parte do sistema poderia mexer nela por fora e pular as regras. Com a cópia somente de leitura, só a conta altera o caderno.

### As regras de negócio (o que o guardião não deixa passar)

**RN01 — Teto de gastos.** Uma despesa nova é bloqueada se o total das despesas ativas passar do limite.
> Exemplo: limite R$ 4.000; já existem R$ 2.320 em despesas ativas. Tentar registrar mais R$ 2.000 daria R$ 4.320, então é **bloqueado**. Vale também ao atualizar o valor de uma despesa.

**RN02 — Cancelado é definitivo.** Um lançamento cancelado não pode mais ser alterado. (Impedir a confirmação de um lançamento cancelado é responsabilidade da classe `LancamentoFinanceiro`, da parte de outro integrante.)

**RN03 — Confirmado não some do histórico.** Um lançamento confirmado não pode ser excluído direto. É preciso **cancelar primeiro** e só então excluir, para o histórico financeiro ficar íntegro.

**RN04 — Valor válido.** Não se registra lançamento com valor menor ou igual a zero. (A regra de e-mail duplicado de usuários fica no cadastro de usuários, em outra parte do projeto.)

**Extra:** também não é permitido registrar dois lançamentos com o **mesmo código**.

### Como a conta reage a erros
Quando uma regra é violada, a conta **interrompe a operação e avisa com uma mensagem clara**, sem alterar nada:

- `IllegalArgumentException`: dado inválido (valor zero, código repetido, código inexistente);
- `IllegalStateException`: operação não permitida no estado atual (estourar o limite, alterar cancelado, excluir confirmado).

### Detalhe técnico importante: o sinal do impacto
`calcularImpactoNoSaldo()` devolve o valor **com sinal**:

- `Despesa` → **negativo** (já somando a taxa adicional);
- `Receita` → **positivo**.

Assim, o saldo é só uma soma: `saldo inicial + impactos dos confirmados`.

---

## 5. `RelatorioFinanceiro` — o resumo

### O que é
Uma classe que só **lê** a conta e imprime relatórios. Ela nunca altera nada, o que a torna segura de usar.

### Os três relatórios

Exemplo usado abaixo: limite R$ 4.000, saldo inicial R$ 1.000, salário de R$ 3.000 (confirmado), aluguel de R$ 1.500 (confirmado, segunda-feira) e mercado de R$ 800 + R$ 20 de taxa (pendente, sábado).

**1. `exibirResumoPorStatus()` — quantos lançamentos em cada situação**
```
Pendente     |   1 lançamento(s) | R$ 820,00
Confirmado   |   2 lançamento(s) | R$ 4.500,00
Cancelado    |   0 lançamento(s) | R$ 0,00
```
> Os valores deste resumo são somados em módulo (receitas e despesas contam como valor positivo), para mostrar o volume movimentado em cada status.

**2. `exibirGastosPorDiaSemana()` — em que dia você mais gasta**
```
Segunda-feira   | R$ 1.500,00
Sábado          | R$ 820,00
(demais dias)   | R$ 0,00
```
> Só entram despesas ativas (as canceladas ficam de fora).

**3. `exibirBalancoGeral()` — a foto completa**
```
Saldo inicial..............: R$ 1.000,00
Receitas confirmadas.......: R$ 3.000,00
Despesas confirmadas.......: R$ 1.500,00
Saldo atual................: R$ 2.500,00
Despesas ativas (c/ pend.).: R$ 2.320,00
Limite de orçamento........: R$ 4.000,00
Uso do limite..............: 58,0%
Orçamento restante.........: R$ 1.680,00
```

---

## 6. Conceitos de POO que aparecem na minha parte

| Conceito | Explicação simples | Onde está |
|---|---|---|
| **Encapsulamento** | Os dados ficam "trancados" (`private`) e só se mexe neles por métodos que validam | `ContaPlanejamento` |
| **Polimorfismo** | A mesma chamada (`calcularImpactoNoSaldo()`) faz coisas diferentes conforme o objeto seja `Despesa` ou `Receita` | O cálculo de saldo e dos relatórios |
| **Enum** | Lista fixa de valores permitidos | `StatusLancamento` |
| **Coleções** | Estruturas para guardar vários objetos (`ArrayList`, `HashMap`) | `ContaPlanejamento` |
| **Agregação** | A conta "tem" lançamentos, mas eles são objetos próprios | Conta ◇ Lançamentos |
| **Tratamento de exceções** | Interromper e avisar com mensagem quando uma regra é quebrada | Regras RN01 a RN04 |

---

## 7. Como minha parte se encaixa no projeto completo

```
Sistema ──usa──▶ Usuario ◆──── ContaPlanejamento ◇──── LancamentoFinanceiro
                                       ▲                      ▲        ▲
                                       │                      │        │
                              RelatorioFinanceiro          Despesa   Receita
```

- O **Usuario** (parte de outro integrante) possui uma `ContaPlanejamento` minha.
- A **Despesa** e a **Receita** (de outro integrante) são os lançamentos que minha conta guarda.
- O **Sistema** (de outro integrante) chama os meus relatórios no menu.

### O que combinei com os colegas
Minhas classes usam estes métodos de `LancamentoFinanceiro`: `getCodigo()`, `getValor()`, `getStatus()`, `getDiaSemana()`, `setValor()`, `setNome()`, `setDescricao()`, `confirmar()`, `cancelar()` e `calcularImpactoNoSaldo()`. O enum `DiaSemana` precisa ter `getDescricao()`. Todas as classes usam o pacote `planejamento`.

---

## 8. Roteiro sugerido para apresentar (cerca de 5 minutos)

1. **Abertura (30 s):** "É um caderninho de finanças digital com teto de gastos. Minha parte é o núcleo: a etiqueta de status, a conta e os relatórios."
2. **`StatusLancamento` (45 s):** mostre os três estados e a ideia de que `CANCELADO` não volta atrás. Explique por que usar enum em vez de texto.
3. **`ContaPlanejamento` (2 min):**
   - explique por que `ArrayList` **e** `HashMap` (ordem × busca rápida);
   - mostre as quatro regras com um exemplo numérico, de preferência o do teto de R$ 4.000 sendo bloqueado;
   - mencione que a conta devolve uma cópia da lista para proteger os dados.
4. **`RelatorioFinanceiro` (1 min):** mostre a saída dos três relatórios e destaque que a classe só lê.
5. **Encaixe (45 s):** mostre o desenho da seção 7 e diga como a minha parte conversa com as dos colegas.
6. **Fechamento (15 s):** "Resultado: as regras de negócio ficam protegidas dentro da conta, e o resto do sistema só precisa usá-la."

**Demonstração rápida (opcional):** registre a receita e as duas despesas do exemplo, tente registrar uma despesa de R$ 2.000 (bloqueada pela RN01) e chame os três relatórios.

---

## 9. Perguntas que podem aparecer e boas respostas

**Por que usar `ArrayList` e `HashMap` juntos?**
O `ArrayList` mantém a ordem e facilita percorrer todos os lançamentos; o `HashMap` permite achar um lançamento direto pelo código. Cada um resolve um problema diferente.

**Onde está o polimorfismo?**
Na lista, que guarda `Despesa` e `Receita` juntas como `LancamentoFinanceiro`. Quando chamo `calcularImpactoNoSaldo()` em cada item, o Java executa a versão correta de cada tipo.

**Por que o relatório não altera a conta?**
Separar quem guarda e protege os dados (conta) de quem apresenta (relatório) deixa o código mais organizado e seguro: um erro no relatório não consegue corromper os dados.

**O que acontece se alguém tenta estourar o limite?**
A conta lança uma exceção com uma mensagem clara e **não registra nada**.

**Por que o cancelamento é separado da exclusão?**
Para preservar o histórico financeiro: o que já foi confirmado precisa ser cancelado (fica registrado) antes de poder sair do caderno.

**Por que `calcularImpactoNoSaldo()` devolve valor negativo para despesas?**
Para o saldo virar uma soma simples e para o cálculo funcionar de forma polimórfica, sem precisar testar o tipo do lançamento em cada passo.

---

## 10. Glossário rápido

| Termo | Significado simples |
|---|---|
| **Classe** | Molde que descreve um tipo de objeto (por exemplo, "conta") |
| **Objeto** | Uma instância concreta criada a partir do molde |
| **Enum** | Lista fixa de valores permitidos |
| **Coleção** | Estrutura que guarda vários objetos (lista, mapa) |
| **`ArrayList`** | Lista ordenada que cresce conforme necessário |
| **`HashMap`** | Estrutura de pares chave → valor, com busca rápida pela chave |
| **Polimorfismo** | Mesma chamada, comportamento diferente conforme o tipo do objeto |
| **Encapsulamento** | Proteger os dados e só permitir acesso por métodos controlados |
| **Exceção** | Aviso de erro que interrompe uma operação inválida |
| **Pacote (`package`)** | "Pasta" lógica que agrupa as classes do projeto |
| **Regra de negócio** | Regra do problema real (por exemplo, "não passar do teto de gastos") |

---

## 11. Como testei

Compilei as três classes junto com versões provisórias das demais e executei um roteiro que exercita cada regra. Resultado:

| Teste | Resultado |
|---|---|
| Registrar receita e despesas dentro do limite | Aceito |
| Despesa que estoura o limite (RN01) | Bloqueada |
| Valor zero (RN04) | Bloqueado |
| Código duplicado | Bloqueado |
| Excluir lançamento confirmado (RN03) | Bloqueado |
| Cancelar e depois excluir (RN03) | Aceito |
| Alterar lançamento cancelado (RN02) | Bloqueado |
| Atualizar despesa acima do limite (RN01) | Bloqueado, com o valor anterior preservado |

Falta apenas a integração final com as classes reais dos outros integrantes.
