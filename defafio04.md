# Questão 4 — Avaliação Técnica do Código: Anti-patterns

## 1. src/config.py

**Classificação:** Anti-pattern

### Configuração carregada no import (linhas 4–7)

**Trecho atual:**

```python
class Config:
    APP_ENV = getenv('APP_ENV', 'local')
    REQUEST_TIMEOUT = int(getenv('REQUEST_TIMEOUT', 10))
```

**Problema:**

A configuração é carregada no momento do import e fica disponível como estado global. Isso dificulta testes e deixa dependências implícitas espalhadas pelo código.

**Sugestão:**

```python
class Config:
    def __init__(self):
        self.APP_ENV = getenv("APP_ENV", "local")
        self.REQUEST_TIMEOUT = int(getenv("REQUEST_TIMEOUT", "10"))
```

---

## 2. src/app.py

**Classificação:** Anti-pattern

### Configuração de CORS muito permissiva (linhas 19–20)

**Trecho atual:**

```python
allow_origins=["*"],
allow_credentials=True,
```

**Problema:**

Essa configuração funciona em desenvolvimento, mas em produção pode abrir brechas de segurança. É importante restringir as origens permitidas com base no ambiente.

**Sugestão:**

```python
origins = ["http://localhost:3000"] if Config.APP_ENV == "local" else ["https://app.suaempresa.com"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
)
```

---

## 3. src/infrastructure/database/sql.py

**Classificação:** Anti-pattern

### Infra acoplada à configuração global (linhas 15–16)

**Trecho atual:**

```python
self._engine = create_async_engine(
    db_url,
    echo=False,
    pool_size=int(Config.SQL_POOL_SIZE),
    max_overflow=int(Config.SQL_MAX_OVERFLOW),
)
```

**Problema:**

A classe PostgresDatabase passa a depender diretamente do Config. Isso traz dois efeitos ruins:

- **Teste fica mais difícil:** você não consegue instanciar o banco com um pool diferente sem mexer em env/config global.
- **A classe fica menos reutilizável:** se amanhã você quiser usar a mesma infra com outra configuração (ou outro microserviço), ela já vem "amarrada" no Config.

**Sugestão:**

Deixar o PostgresDatabase receber tudo por parâmetro e o Container montar:

```python
class PostgresDatabase(IDatabase):
    def __init__(self, db_url: str, pool_size: int, max_overflow: int):
        self._engine = create_async_engine(
            db_url, 
            pool_size=pool_size, 
            max_overflow=max_overflow
        )
```

---

## 4. src/infrastructure/repositories/sql_repository.py

**Classificação:** Anti-pattern

### `model = Base` no repository base (linhas 7–8)

**Trecho atual:**

```python
class SqlRepository(IRepository):
    model = Base
```

**Problema:**

`Base` é só a classe base dos modelos, não uma tabela. Defini-la como model padrão pode levar ao uso incorreto do repository.

O risco aqui é que alguém esqueça de setar model na subclasse e o repo tente executar select(Base), gerando comportamento inválido e confuso. 

**Sugestão:**

Forçar o contrato: repo base não tem model, quem herda define.

```python
class SqlRepository(IRepository):
    model = None  # subclasses definem

    def __init__(self, session_factory):
        super().__init__(session_factory)
        if self.model is None:
            raise ValueError(
                f"{self.__class__.__name__} deve definir o atributo 'model'"
            )
```

---

**Classificação:** Anti-pattern

### Commit dentro do repository (linhas 25–30)

**Trecho atual:**

```python
async def create(self, values):
    async with self.session_factory() as session:
        _model = self.model(**values)
        session.add(_model)
        await session.commit()
        return _model
```

**Problema:**

Quando o commit() é executado dentro do repository, ele passa a controlar o encerramento da transação sem conhecer o fluxo completo da operação.

Em cenários que envolvem múltiplas etapas (ex.: criar customer, criar permissões e registrar auditoria), um erro intermediário pode deixar o sistema inconsistente, pois parte das operações já terá sido persistida.

O controle transacional deve ficar no service/use case, que conhece todo o fluxo e pode decidir quando efetivamente finalizar (commit) ou reverter (rollback) a transação.

**Sugestão:**

Repository só adiciona; service decide commit:

```python
# repository
async def create(self, session, values):
    obj = self.model(**values)
    session.add(obj)
    return obj
```

---

## 5. src/middlewares/tools.py

**Classificação:** Anti-pattern

### Classe Tools concentrando responsabilidades e herdando repository (linhas 12–51)

**Trecho atual:**

```python
class Tools(SqlRepository):
    # implementação
```

**Problema:**

A classe mistura integração HTTP, acesso a banco, validação e ainda herda de repository apenas para reaproveitar sessão. Isso gera acoplamento forte e dificulta testes e evolução.

**Sugestão:**

Separar responsabilidades em classes distintas (HttpClient, SecretRepository, Validator) e compor no service quando necessário.

---

**Classificação:** Anti-pattern + Bug

### SQL Injection via f-string (linhas 32)

**Trecho atual:**

```python
await session.execute(f"SELECT access_key FROM secrets WHERE id = {id}")
```

**Problema:**

Interpolar SQL com f-string é perigoso e pode quebrar a query ou abrir brecha de segurança. Sempre use parametrização para evitar SQL injection.

**Sugestão:**

```python
from sqlalchemy import text

result = await session.execute(
    text("SELECT access_key FROM secrets WHERE id = :id"),
    {"id": id}
)
```

---

**Classificação:** Anti-pattern

### Criação de `httpx.AsyncClient` por chamada (linhas 17–26)

**Trecho atual:**

```python
async def send_instant_message(self, msg_content: str):
    async with httpx.AsyncClient() as client:
        body = {'message': msg_content}
        request = await client.post(
            f"{self.chat_base_url}/channel/webhook",
            headers=self.headers,
            json=body,
            timeout=self.request_timeout
        )

```

**Problema:**

Criar o client a cada chamada gera overhead desnecessário. É mais eficiente reutilizar uma instância do client, criando-a uma vez e reutilizando-a ao longo do ciclo de vida da aplicação.

**Sugestão:**

```python
class HttpService:
    def __init__(self):
        self.client = httpx.AsyncClient()
    
    async def send_instant_message(self, msg_content: str):
    return await self._client.post(...)
```

---

## 6. src/middlewares/exception_handler.py

**Classificação:** Anti-pattern

### Captura genérica de exceções e retorno fixo 500 (linhas 11–14)

**Trecho atual:**

```python
except Exception as exception:
    return JSONResponse(status_code=500, ...)
```

**Problema:**

Isso mascara erros esperados (400, 404) e dificulta observabilidade. É importante diferenciar exceções conhecidas e tratá-las de forma apropriada, permitindo que o FastAPI lide com HTTPException.

**Sugestão:**

```python
from fastapi import HTTPException

try:
    # código
except HTTPException:
    raise  # deixa o FastAPI tratar
except ValidationError as e:
    return JSONResponse(status_code=400, content={"error": str(e)})
except Exception as e:
    logger.error(f"Erro inesperado: {e}", exc_info=True)
    return JSONResponse(status_code=500, content={"error": "Internal server error"})
```

---

**Classificação:** Anti-pattern

### Vazamento de stack trace para o cliente (linhas 13–15)

**Trecho atual:**

```python
content_error = f'Exception: {exception} - StackTrace: {trace}'
```

**Problema:**

Stack trace deve ser logado internamente, não enviado no response. Expor detalhes de erro internos pode revelar informações sensíveis sobre a aplicação e facilitar ataques.

**Sugestão:**

```python
logger.error(f"Exception: {exception}", exc_info=True)
return JSONResponse(
    status_code=500, 
    content={"error": "Internal server error"}
)
```

---

## 7. src/endpoints/health/controller.py

**Classificação:** Anti-pattern

### Resposta inconsistente no endpoint `/health` (linhas 22–25)

**Trecho atual:**

```python
if not all(statuses.values()):
    return JSONResponse(status_code=503, content=statuses)

return statuses
```

**Problema:**

O endpoint retorna formatos diferentes dependendo do fluxo, o que quebra a consistência do contrato. É importante manter o mesmo formato de resposta independente do status code.

**Sugestão:**

```python
status_code = 503 if not all(statuses.values()) else 200
return JSONResponse(status_code=status_code, content=statuses)
```

---

**Classificação:** Anti-pattern

### Uso manual de `JSONResponse` no endpoint `/beat` (linhas 15)

**Trecho atual:**

```python
return JSONResponse(status_code=status.HTTP_200_OK, content="OK")
```

**Problema:**

Nesse caso, retornar um dict é suficiente e mais simples. O FastAPI automaticamente serializa dicts em JSON com status 200.

**Sugestão:**

```python
return {"status": "OK"}
```

---

## 8. src/endpoints/health/service.py

**Classificação:** Anti-pattern

### Falha silenciosa no healthcheck (linhas 14–18)

**Trecho atual:**

```python
try:
    await self._repository.check_health()
    return True
except Exception:
    return False
```

**Problema:**

A falha é escondida e a causa do problema se perde. É importante logar a exceção para facilitar diagnóstico de problemas em produção.

**Sugestão:**

```python
try:
    await self._repository.check_health()
    return True
except Exception as e:
    logger.error(f"Health check failed: {e}", exc_info=True)
    return False
```

---

## 9. src/endpoints/health/repository.py

**Classificação:** Anti-pattern

### Execução de SQL sem padronização (linhas 7)

**Trecho atual:**

```python
await session.execute('SELECT NOW()')
```

**Problema:**

Padronizar com `text()` deixa mais claro e consistente. Além disso, ajuda a evitar problemas de compatibilidade com diferentes dialetos SQL.

**Sugestão:**

```python
from sqlalchemy import text

await session.execute(text('SELECT NOW()'))
```

---


## 10. src/endpoints/registration/manager.py

**Classificação:** Anti-pattern

### Orchestrator ignorando o service (linhas 10)

**Trecho atual:**

```python
return await self._repository.filter_by({'id': customer_id})
```

**Problema:**

O orchestrator pula o service e ignora regras de negócio já definidas. Isso quebra a separação de responsabilidades e pode fazer com que validações importantes sejam ignoradas.

**Sugestão:**

```python
return await self._service.get_customer(customer_id)
```

---

## 11. src/endpoints/registration/service.py

**Classificação:** Anti-pattern

### Regra de negócio definida, mas não aplicada (linhas 8–12)

**Trecho atual:**

```python
if not customer:
    raise CustomerNotFoundException(...)
```

**Problema:**

Essa regra pode nunca ser executada no fluxo atual, pois o orchestrator acessa diretamente o repository. É importante garantir que todo acesso passe pelo service, onde as regras de negócio são aplicadas.

**Sugestão:**

Garantir que todo acesso passe pelo service, onde as regras de negócio são aplicadas.
