# Otimização de Estacionamento de Aviões com OR-Tools(optimiza_ia_matriz_tempo)
Este projeto visa resolver o problema de otimização do estacionamento de aviões em um aeroporto, utilizando a biblioteca ortools.sat.python.cp_model do Google OR-Tools. A seguir, está uma descrição detalhada das funções e classes utilizadas no código.

## Requisitos
<!-- ortools, na versão 9.7.2996 -->
- Python 3.6 ou superior
- Bibliotecas: `ortools` -> versão 9.7.2996

## Código
### Importações e Configurações
```python
from ortools.sat.python import cp_model
```
Funções
1. `resolve(solucionador, modelo, X)`
Esta função resolve o modelo de otimização e exibe o status e os resultados.

```python
def resolve(solucionador, modelo, X):
    status = solucionador.Solve(modelo)
    print(f"Status: {status}")

    if status == cp_model.OPTIMAL:
        print("Optimal")
        for i, matriz_estacio_tempo in enumerate(X):
            for j, tempos in enumerate(matriz_estacio_tempo):
                for k, variavel in enumerate(tempos):
                    valor = solucionador.Value(variavel)
                    if valor == 1:
                        print(f"O aviao {i} no estacionamento {j} no tempo {k}")
    else:
        print("Solução não encontrada")
```    
## Definindo o Modelo
```python
modelo = cp_model.CpModel()
solucionador = cp_model.CpSolver()

total_avioes = 3
total_estacios = 2 
total_tempo = 5
momento_de_chegada = [1, 2, 0]
tempo_duracao = [3, 3, 2]

X = [[[modelo.NewBoolVar(f"aviao_{i}_estacionamento_{j}_no_tempo_{k}") for k in range(total_tempo)] for j in range(total_estacios)] for i in range(total_avioes)]

custos = [
    [1000, 500],
    [1000, 500],
    [1000, 500]
]

Y = []
```
## Restrições
Restrição 1: Cada avião deve estar em pelo menos um lugar
```python
for i in range(total_avioes):
    posicoes_do_aviao = [X[i][j][k] for j in range(total_estacios) for k in range(total_tempo)]
    modelo.Add(sum(posicoes_do_aviao) >= 1)
```    
Restrição 2: Apenas um avião por estacionamento em cada tempo
```python
for j in range(total_estacios):
    for k in range(total_tempo):
        avioes_agora = [X[i][j][k] for i in range(total_avioes)]
        modelo.Add(sum(avioes_agora) <= 1)
```        
Restrição 3: Controle de presença dos aviões em diferentes estacionamentos
```python
for i in range(total_avioes):
    linha_aviao_i = []
    for j in range(total_estacios):
        aviao_i_em_j = modelo.NewBoolVar(f"aviao_{i}_estacionamento_{j}")
        linha_aviao_i.append(aviao_i_em_j)
        variaveis_aviao_estacionamento = [X[i][j][k] for k in range(total_tempo)]

        modelo.Add(sum(variaveis_aviao_estacionamento) > 0).OnlyEnforceIf(aviao_i_em_j)        
        modelo.Add(sum(variaveis_aviao_estacionamento) == 0).OnlyEnforceIf(aviao_i_em_j.Not()) 

        aviao_i_em_outro_sem_ser_j = modelo.NewBoolVar(f"aviao_{i}_em_outro_sem_ser_{j}")       
        variaveis_outros_estacios = [X[i][outro_estacio][k] for k in range(total_tempo) for outro_estacio in range(total_estacios) if outro_estacio != j]

        modelo.Add(sum(variaveis_outros_estacios) > 0).OnlyEnforceIf(aviao_i_em_outro_sem_ser_j)
        modelo.Add(sum(variaveis_outros_estacios) == 0).OnlyEnforceIf(aviao_i_em_outro_sem_ser_j.Not())

        modelo.AddImplication(aviao_i_em_j, aviao_i_em_outro_sem_ser_j.Not())
    Y.append(linha_aviao_i)
```
Restrição 4: Tempo de duração do estacionamento
```python
for i in range(total_avioes):
    tudo_aviao = [X[i][j][k] for j in range(total_estacios) for k in range(total_tempo)]
    modelo.Add(sum(tudo_aviao) == tempo_duracao[i])
```   
Restrição 5: Controle de pouso e decolagem
```python
for i in range(total_avioes):
    for j in range(total_estacios):
        for k in range(total_tempo):
            if k > 0 and k < total_tempo - 1:
                se_aviao_decolou = modelo.NewBoolVar(f"aviao_{i}_est_{j}_tempo_{k}_decolou_agora")
                modelo.Add(sum([X[i][j][k-1], X[i][j][k].Not()]) == 2).OnlyEnforceIf(se_aviao_decolou)
                modelo.Add(sum([X[i][j][k-1], X[i][j][k].Not()]) != 2).OnlyEnforceIf(se_aviao_decolou.Not())

                for f in range(k+1, total_tempo):
                    modelo.AddImplication(se_aviao_decolou, X[i][j][f].Not())    

            if k < total_tempo - 1:
                se_aviao_pousou = modelo.NewBoolVar(f"aviao_{i}_est_{j}_tempo_{k}_pousou_agora")
                modelo.Add(sum([X[i][j][k+1], X[i][j][k].Not()]) == 2).OnlyEnforceIf(se_aviao_pousou)
                modelo.Add(sum([X[i][j][k+1], X[i][j][k].Not()]) != 2).OnlyEnforceIf(se_aviao_pousou.Not())

                for p in range(0, k):
                    modelo.AddImplication(se_aviao_pousou, X[i][j][p].Not())
```                    
Restrição 6: Controle de momento de chegada
```python
for i in range(total_avioes):
    momento_de_chegada_dele = momento_de_chegada[i]
    for j in range(total_estacios):
        for k in range(momento_de_chegada_dele):
            modelo.Add(X[i][j][k] == 0)

for i in range(total_avioes):
    momento_de_chegada_dele = momento_de_chegada[i]
    todos_os_estacios = [X[i][j][momento_de_chegada_dele] for j in range(total_estacios)]
    modelo.Add(sum(todos_os_estacios) == 1)
```    
## Função Objetivo
```python
super_custo = [custos[i][j] * Y[i][j] for i in range(total_avioes) for j in range(total_estacios)]
modelo.Minimize(sum(super_custo))
```
## Resolução do Modelo
```python
resolve(solucionador, modelo, X)
```
## Conclusão
Este projeto demonstra como utilizar OR-Tools para resolver um problema complexo de otimização envolvendo a alocação de aviões em locais de estacionamento ao longo do tempo, levando em conta diversas restrições e preferências.






# Otimização de Estacionamento de Aviões com OR-Tools(optimiza_ia)
Este projeto visa resolver o problema de otimização do estacionamento de aviões em um aeroporto, utilizando a biblioteca ortools.sat.python.cp_model do Google OR-Tools. A seguir, está uma descrição detalhada das funções e classes utilizadas no código.
<!-- ortools, na versão 9.7.2996 -->

## Descrição das Funções
1. `tamanho(modelo, estacios, avioes)`
Garante que aviões grandes só possam estacionar em locais apropriados.

2. `todos_estacionar(total_de_avioes, estacios, modelo)`
Assegura que todos os aviões tenham um local para estacionar, utilizando variáveis booleanas.

3. `destintos(estacios, modelo)`
Garante que cada local de estacionamento receba um avião distinto.

4. `requerem_passaporte(modelo, estacios, avioes)`
Impedir que aviões que requerem passaporte estacionem em locais que não oferecem controle de passaporte.

5. `prefere_com_passporte(modelo, estacios, avioes)`
Adiciona penalidades para aviões que preferem estacionar em locais com controle de passaporte, mas não é obrigatório.

6. `limita_vizinhos(modelo, estacios, avioes)`
Restringe a alocação de aviões grandes em estacionamentos vizinhos.

7. `resolver(solucionador, modelo, estacios, avioes, penalidades)`
Resolve o modelo de otimização e exibe os resultados.

## Classes
1. `estacio`
Representa um local de estacionamento no aeroporto.

### Atributos:
- `grande`: Indica se o local pode acomodar aviões grandes.
- `tem_passaporte`: Indica se o local tem controle de passaporte.
- `variavel`: Variável do modelo que representa o local de estacionamento.
- `k`: Identificador do local de estacionamento.
- `vizinhos`: Lista de locais vizinhos.
- `recebe_grande`: Variável booleana indicando se o local recebe aviões grandes.

2. `aviao`
Representa um avião que precisa de um local para estacionar.

### Atributos:
- `k`: Identificador do avião.
- `grande`: Indica se o avião é grande.
- `requer_passaporte`: Indica se o avião requer controle de passaporte.

## Execução
Para executar o projeto, siga os passos abaixo:

1. Crie um modelo de otimização:

```python
modelo = cp_model.CpModel()
```
2. Defina os locais de estacionamento e aviões:

```python
estacios = [estacio(k, total_de_avioes, grande, modelo, tem_passaporte) for k in range(num_estacios)]
avioes = [aviao(k, grande, requer_passaporte) for k in range(num_avioes)]
```
3. Adicione as restrições ao modelo:

```python
tamanho(modelo, estacios, avioes)
todos_estacionar(total_de_avioes, estacios, modelo)
destintos(estacios, modelo)
requerem_passaporte(modelo, estacios, avioes)
penalidades = prefere_com_passporte(modelo, estacios, avioes)
limita_vizinhos(modelo, estacios, avioes)
```
4. Resolva o modelo:

```python
solucionador = cp_model.CpSolver()
resolver(solucionador, modelo, estacios, avioes, penalidades)
```
Conclusão
Este projeto demonstra como utilizar OR-Tools para resolver um problema complexo de otimização envolvendo a alocação de aviões em locais de estacionamento, levando em conta diversas restrições e preferências.
usando de uma logica mais computacional comum






