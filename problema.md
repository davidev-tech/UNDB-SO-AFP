## Mapa causal do problema

### 1. Raízes (causas fundamentais)

São as falhas estruturais que permitem os sintomas aparecerem:

| Raiz | Explicação técnica | Evidência |
|---|---|---|
| **Ausência de atomicidade no estoque** | Leitura e escrita não são uma operação única. Dois processos leem o mesmo saldo e depois subtraem, gerando *lost update*. | PID_01 e PID_02 leem estoque = 1 e ambos escrevem -1. |
| **Falta de exclusão mútua** | Não há mutex, lock de linha, transação ou semáforo protegendo a região crítica do produto. | Anexo I, 10:01:05–10:01:08. |
| **Aquisição de locks em ordem inconsistente** | PID_15 pega Lock_Inventario e espera Lock_Financeiro; PID_16 pega Lock_Financeiro e espera Lock_Inventario. Isso forma um ciclo de espera. | Anexo I, 14:02:10–14:02:13. |
| **Ausência de timeout/try_lock** | Processos ficam bloqueados indefinidamente aguardando recurso, sem rollback ou desistência. | PID_15 e PID_16 em estado *Waiting*. |
| **Escalonamento sem aging ou prioridade efetiva** | Processos de baixa prioridade não recebem fatia de CPU porque compras dominam a fila. | Sintoma relatado: relatórios noturnos nunca executam. |
| **Modelo de concorrência bloqueante** | Workers ficam presos em validações financeiras, sem assincronismo, fila ou timeout. | Regra 2: compra não pode travar mais de 5s. |

---

### 2. Consequências (sintomas técnicos observáveis)

| Consequência | Descrição | Regra violada |
|---|---|---|
| **Estoque negativo** | Dois processos compram o mesmo item e o saldo final fica -1. | Regra 1 – Integridade |
| **Deadlock** | PID_15 e PID_16 ficam presos esperando recursos um do outro. | Regra 2 – Liveness |
| **Travamento indefinido** | Processos aguardam validação financeira sem limite de tempo. | Regra 2 |
| **Starvation** | Relatórios de baixa prioridade nunca recebem CPU. | Regra 3 – Fairness |
| **Inconsistência de dados** | O banco passa a refletir compras que não deveriam existir. | Regra 1 |
| **Perda de confiabilidade** | O sistema não resiste ao tráfego real. | Requisito do cliente |

---

### 3. Impactos

| Dimensão | Impacto |
|---|---|
| **Negócio** | Venda de itens sem estoque, cancelamentos, prejuízo financeiro e perda de confiança do cliente. |
| **Operacional** | Relatórios atrasados, auditoria comprometida e decisões baseadas em dados errados. |
| **Técnico** | Sistema instável, travamentos, deadlocks, retrabalho e dificuldade de escalar. |
| **Avaliação** | Não atende às regras dos Anexos I e II; falha na Defesa Técnica por não explicar e corrigir as falhas. |
| **Cliente** | Recebe um sistema que não funciona com o tráfego real da operação. |

---

### 4. Leitura direta dos logs

- **10:01:05–10:01:08**  
  PID_01 e PID_02 leem estoque = 1. Ambos subtraem 1. O segundo WRITE gera estoque = -1.  
  **Raiz:** condição de corrida no tipo *check-then-act*.  
  **Consequência:** estoque negativo.  
  **Impacto:** venda fantasma e inconsistência no banco.

- **14:02:10–14:02:13**  
  PID_15 adquire Lock_Inventario e espera Lock_Financeiro. PID_16 adquire Lock_Financeiro e espera Lock_Inventario.  
  **Raiz:** ordem circular de aquisição de locks.  
  **Consequência:** deadlock.  
  **Impacto:** travamento do servidor e violação da Regra 2.

- **Processos de baixa prioridade**  
  Não aparecem executando nos logs.  
  **Raiz:** escalonador sem aging/time slice garantido.  
  **Consequência:** starvation.  
  **Impacto:** relatórios noturnos nunca saem.

---

### 5. Ajustes técnicos fundamentados

1. **Atomicidade no estoque**  
   Usar transação com `SELECT ... FOR UPDATE` ou `UPDATE produto SET estoque = estoque - 1 WHERE id = ? AND estoque >= 1`.  
   Se `affected_rows = 0`, a compra falha por falta de estoque.

2. **Exclusão mútua**  
   Aplicar mutex/semáforo por produto ou lock de linha no banco. Para leitura e escrita no mesmo item, a operação deve ser atômica.

3. **Prevenção de deadlock**  
   Definir ordem global de locks: por exemplo, sempre adquirir `Lock_Inventario` antes de `Lock_Financeiro`. Usar `try_lock` com timeout e rollback.

4. **Liveness**  
   Todo lock deve ter timeout. A validação financeira não pode bloquear a compra por mais de 5 segundos. Se estourar, rollback e resposta de erro.

5. **Fairness**  
   Usar escalonamento por prioridade com *aging*: relatórios de baixa prioridade ganham prioridade conforme o tempo de espera. Garantir fatia de CPU a cada 10 minutos.

6. **Assincronismo**  
   Não bloquear worker principal em rotinas de background. Usar filas, thread pool separada ou processamento assíncrono.

Em resumo: a raiz está na **falta de controle de concorrência** e no **escalonamento inadequado**; as consequências são **estoque negativo, deadlock e starvation**; os impactos atingem **integridade dos dados, disponibilidade, cliente e avaliação técnica**.
