# Questão 6 — Code Review Bot

## 1. bot.py

### Organização dos imports (linhas 2–4)

#### Trecho atual

```python
import os, sys, traceback, logging, configparser
import xlsxwriter
from datetime import datetime, timedelta, timezone
```

#### Sugestão

Separar os imports por linha e organizá-los por categoria (biblioteca padrão e dependências externas), conforme recomendado pela PEP 8.

Além disso, remover imports que não estão sendo utilizados (`traceback`, `timedelta`, `timezone`) ajuda a manter o código mais legível, reduz ruído cognitivo e evita dúvidas durante manutenção ou revisões futuras.

#### Exemplo

```python
import os
import sys
import logging
import configparser
from datetime import datetime

import xlsxwriter
from apscheduler.schedulers.blocking import BlockingScheduler
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from logging.handlers import RotatingFileHandler
```

---

### Configuração de credenciais diretamente no código (linha 19)

#### Trecho atual

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql+psycopg2://postgres:123mudar@127.0.0.1:5432/bot_db'
```

#### Sugestão

Externalizar a string de conexão para uma variável de ambiente.

Isso evita exposição de credenciais no repositório e facilita a execução do bot em diferentes ambientes (local, CI, produção).

#### Exemplo

```python
database_url = os.getenv("DATABASE_URL")
if not database_url:
    raise RuntimeError("DATABASE_URL não configurada")

app.config["SQLALCHEMY_DATABASE_URI"] = database_url
```

---

### Caminho do arquivo de configuração fixo (linha 22)

#### Trecho atual

```python
config.read('/tmp/bot/settings/config.ini')
```

#### Sugestão

Permitir que o caminho do arquivo seja configurável via variável de ambiente e validar se o arquivo foi carregado corretamente.

Isso torna o código mais portátil e reduz falhas silenciosas em ambientes distintos.

#### Exemplo

```python
config_path = os.getenv("BOT_CONFIG_PATH", "/tmp/bot/settings/config.ini")
read_files = config.read(config_path)

if not read_files:
    raise RuntimeError(f"Arquivo de configuração não encontrado em {config_path}")
```

---

### Nome da variável e validação do intervalo do scheduler (linhas 24-25)

#### Trecho atual

```python
var1 = int(config.get('scheduler','IntervalInMinutes'))
app.logger.warning('Intervalo entre as execucoes do processo: {}'.format(var1))
```

#### Sugestão

Usar um nome de variável mais descritivo melhora a leitura do código.

Também é interessante validar o valor lido e utilizar o nível de log `info`, já que se trata de um comportamento esperado da aplicação.

#### Exemplo

```python
interval_minutes = config.getint("scheduler", "IntervalInMinutes")

if interval_minutes <= 0:
    raise RuntimeError("IntervalInMinutes deve ser maior que zero")

app.logger.info("Intervalo entre execuções (min): %s", interval_minutes)
```

---

### Agendamento do job no APScheduler (linha 28)

#### Trecho atual

```python
scheduler.add_job(task1(db), 'interval', id='task1_job', minutes=var1)
```

#### Sugestão

Passar a função como referência e utilizar `args` para injetar dependências.

Isso garante que o job seja executado apenas pelo scheduler, respeitando o intervalo configurado.

#### Exemplo

```python
scheduler.add_job(
    task1,
    trigger="interval",
    id="task1_job",
    minutes=interval_minutes,
    args=[db],
    replace_existing=True,
)
```

---

### Uso de print em vez de logging (linhas 13 e 79)

#### Trechos atuais

```python
print('Press Crtl+{0} to exit'.format('Break' if os.name == 'nt' else 'C'))
print('job executed!')
```

#### Sugestão

Padronizar o uso de logging melhora a observabilidade, facilita depuração e integração com ferramentas de monitoramento.

#### Exemplo

```python
app.logger.info("Bot iniciado. Ctrl+C para encerrar.")
app.logger.info("Job de exportação finalizado com sucesso")
```

---

### Consulta SQL genérica (SELECT *) (linha 48)

#### Trecho atual

```python
orders = db.session.execute('SELECT * FROM users;')
```

#### Sugestão

Selecionar explicitamente apenas as colunas necessárias melhora performance e reduz o acoplamento com o schema do banco, além de tornar o código mais resiliente a mudanças futuras.

#### Exemplo

```python
sql = """
SELECT id, name, email, role_id, created_at, updated_at
FROM users
ORDER BY id
"""
users_rows = db.session.execute(sql)
```

---

### Exportação de dados sensíveis (linhas 55 e 70)

#### Trecho atual

```python
worksheet.write('D{0}'.format(index),'Password')
worksheet.write('D{0}'.format(index),order[3])
```

#### Sugestão

Evitar exportar campos sensíveis como senha, mesmo quando armazenados de forma hasheada.

Isso reduz riscos de exposição indevida e segue boas práticas de segurança e privacidade.

---

### Garantia de fechamento do arquivo de exportação (linhas 45-78)

#### Trecho atual

```python
workbook.close()
```

#### Sugestão

Garantir o fechamento do arquivo mesmo em caso de exceção evita arquivos corrompidos e facilita reexecuções do job.

#### Exemplo

```python
workbook = xlsxwriter.Workbook(file_path)
try:
    worksheet = workbook.add_worksheet()
    # escrita dos dados
finally:
    workbook.close()
```

---

## 2. Arquivos de suporte

### Pipfile.lock — versão do Python

**Local:** `_meta.requires.python_version`

```json
"python_version": "3.7"
```

#### Sugestão

Avaliar a atualização do runtime para uma versão mais recente do Python, o que pode facilitar upgrades de dependências e melhorias de segurança.

---

### Pipfile.lock — versões das dependências

**Local:** seção `default`

**Exemplos:**

- `Flask==1.1.2`
- `SQLAlchemy==1.3.18`
- `psycopg2==2.8.5`

#### Sugestão

Planejar um upgrade gradual das dependências ao longo do tempo. Versões mais antigas podem limitar a evolução do projeto e correções já disponíveis em releases mais recentes.

---

### Pipfile

**Local:** seção `[packages]`

```toml
logging = "*"
```

#### Sugestão

Remover `logging`, pois trata-se de um módulo da biblioteca padrão do Python.

---

**Local:** dependências com `*`

```toml
apscheduler = "*"
flask = "*"
...
```

#### Sugestão

Definir faixas básicas de versão ajuda a tornar mais explícita a expectativa do ambiente e facilita upgrades controlados.

---

### settings/config.ini

#### Conteúdo

```ini
[scheduler]
IntervalInMinutes: 60
```

#### Sugestão

O arquivo está simples e claro. Como melhoria, é importante garantir no código a validação da existência e do valor dessa configuração.

Opcionalmente, pode-se adotar uma padronização de nomenclatura para facilitar leitura e manutenção.

#### Exemplo opcional

```ini
[scheduler]
interval_in_minutes = 60
```
