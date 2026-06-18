# Sistema de Agendamento de Tarefas Internas

Sistema distribuído composto por dois microsserviços Spring Boot que gerenciam **funcionários** e suas **atividades/tarefas internas**. O microsserviço de atividades se comunica com o de funcionários via API REST antes de cadastrar qualquer tarefa, garantindo que o funcionário responsável exista e esteja ativo no sistema.

---

## Tema do Sistema

O sistema permite que uma empresa gerencie seus funcionários e atribua tarefas internas a eles. Antes de cadastrar uma atividade, o sistema verifica automaticamente se o funcionário existe e está com status **ATIVO**, garantindo integridade nos dados e regras de negócio consistentes entre os serviços.


## Como Executar os Microsserviços

### Pré-requisitos

- Java 17 ou superior instalado
- Maven 3.8 ou superior instalado e configurado no PATH

### Passo a Passo

> **Atenção:** O microsserviço de **funcionários deve ser iniciado primeiro**, pois o de atividades depende dele para funcionar corretamente.

**Terminal 1 — Microsserviço de Funcionários:**
```bash
cd microsservico-funcionarios
mvn spring-boot:run
```

**Terminal 2 — Microsserviço de Atividades (somente após o Terminal 1 estar UP):**
```bash
cd microsservico-atividades
mvn spring-boot:run
```

## Portas Utilizadas

| Microsserviço       | Porta | Swagger UI                              | Console H2                              |
|---------------------|-------|-----------------------------------------|-----------------------------------------|
| Funcionários        | 8081  | http://localhost:8081/swagger-ui.html   | http://localhost:8081/h2-console        |
| Atividades          | 8082  | http://localhost:8082/swagger-ui.html   | http://localhost:8082/h2-console        |

### Acesso ao Console H2

| Campo     | Funcionários                    | Atividades                     |
|-----------|---------------------------------|--------------------------------|
| JDBC URL  | `jdbc:h2:mem:funcionariosdb`    | `jdbc:h2:mem:atividadesdb`     |
| User Name | `sa`                            | `sa`                           |
| Password  | *(deixar em branco)*            | *(deixar em branco)*           |

---

## Endpoints Principais

### Microsserviço de Funcionários — porta 8081

| Método   | Endpoint                    | Descrição                                         |
|----------|-----------------------------|---------------------------------------------------|
| GET      | `/funcionarios`             | Lista todos os funcionários                       |
| GET      | `/funcionarios/{id}`        | Busca funcionário por ID                          |
| POST     | `/funcionarios`             | Cadastra novo funcionário                         |
| PUT      | `/funcionarios/{id}`        | Atualiza funcionário existente                    |
| DELETE   | `/funcionarios/{id}`        | Remove funcionário                                |
| GET      | `/funcionarios/{id}/existe` | Verifica existência (uso interno entre serviços)  |

**Status disponíveis:** `ATIVO`, `INATIVO`, `FERIAS`, `AFASTADO`

---

### Microsserviço de Atividades — porta 8082

| Método   | Endpoint                    | Descrição                                         |
|----------|-----------------------------|---------------------------------------------------|
| GET      | `/atividades`               | Lista todas as atividades                         |
| GET      | `/atividades/{id}`          | Busca atividade por ID                            |
| POST     | `/atividades`               | Cadastra nova atividade                           |
| PUT      | `/atividades/{id}`          | Atualiza atividade existente                      |
| DELETE   | `/atividades/{id}`          | Remove atividade                                  |
| PATCH    | `/atividades/{id}/status`   | Atualiza apenas o status da atividade             |
| GET      | `/atividades?funcionarioId` | Lista atividades de um funcionário específico     |
| GET      | `/atividades?status`        | Lista atividades filtradas por status             |

**Status disponíveis:** `PENDENTE`, `EM_ANDAMENTO`, `CONCLUIDA`, `CANCELADA`

**Prioridades disponíveis:** `BAIXA`, `MEDIA`, `ALTA`, `CRITICA`

---

## Comunicação Entre os Microsserviços

Ao realizar um `POST /atividades`, o microsserviço de atividades (porta **8082**) faz uma chamada HTTP GET para o microsserviço de funcionários (porta **8081**) antes de salvar a atividade:

```
[Cliente] --POST /atividades--> [Microsserviço Atividades :8082]
                                         |
                          GET /funcionarios/{funcionarioId}
                                         |
                                         v
                              [Microsserviço Funcionários :8081]
                                         |
                              Retorna dados do funcionário
                                         |
                                         v
                         Valida se funcionário existe e está ATIVO
                                         |
                                         v
                              Salva atividade no banco
```

**Regras de negócio validadas na comunicação:**
- O funcionário informado no `funcionarioId` deve existir no microsserviço de funcionários
- O funcionário deve estar com status `ATIVO` para receber novas tarefas
- Se o microsserviço de funcionários estiver indisponível, retorna erro `503 Service Unavailable`
- Se o funcionário não for encontrado, retorna erro `422 Unprocessable Entity`
