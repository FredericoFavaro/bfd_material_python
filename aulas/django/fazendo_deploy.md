# Fazendo deploy

**Fazer deploy** significa implantar um software ou site, movendo-o do ambiente de desenvolvimento para um servidor, tornando-o acessível ao público ou em um ambiente de testes/produção

No caso, vamos fazer o deploy de nosso blog para o [Render](https://render.com/), uma plataforma que da suporte a banco de dados (PostgreSQL) e ao Django na tier/plano gratuito. Existem outras plataformas que fornecem o mesmo serviço, algumas com limitações diferentes.

A configuração da aplicação os procedimentos usados para fazer o deploy, pode variar consideravelmente de uma plataforma para outra, por isso é sempre importante dar uma olhada na documentação da plataforma bem como, checar os planos disponíveis, que podem variam com o tempo.

A documentação do Render para deploy de uma aplicação Django está disponível no link: [Deploy a Django App on Render](https://render.com/docs/deploy-django)
  
Vamos começar?
  
## Pré-Requisitos

- Aplicação Django disponível no GitHub, GitLab, GitBucket e etc.
- Ter uma conta no [Render](https://render.com/).
## Configurando a aplicação

Primeiro vamos fazer alguns ajustes, para que sua aplicação Django consiga funcionar corretamente no servidor.

### Configurando o banco de dados

- Instalar os pacotes **psycopg2** e **DJ-Database-URL** na venv do projeto: `pip install psycopg2-binary, dj-database-url`
- No arquivo settings.py:
	- Importe: `from dj_database_url import parse as dburl`
- Em DATABASES, apague e adicione:
```python
default_dburl = 'sqlite:///' + str(BASE_DIR / 'db.sqlite3')

DATABASES = {
	'default': config('DATABASE_URL', default=default_dburl, cast=dburl),
}
```

### Ajustando os arquivos static

- Instale o pacote whitenoise com a dependencia brotli na venv do projeto: `pip install 'whitenoise[brotli]'`.
- No arquivo settings.py:
	- Vá para MIDDLEWARE e adicione a linha `'whitenoise.middleware.WhiteNoiseMiddleware',` imediatamente depois da linha `'django.middleware.security.SecurityMiddleware',`.
	- Vá para a sessão de configuração de arquivos static e adicione:
```python
STATIC_URL = '/static/'

if not DEBUG:
    STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
    STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
else:
    STATIC_ROOT = BASE_DIR / 'static'
```

### Preparando tudo para produção
- Crie o arquivo `build.sh` na raiz do projeto e adicione o conteúdo:
```bash
#!/usr/bin/env bash
# Exit on error
set -o errexit

# Modify this line as needed for your package manager (pip, poetry, etc.)
pip install -r requirements.txt

# Convert static asset files
python manage.py collectstatic --no-input

# Apply any outstanding database migrations
python manage.py migrate

# If the CRESTE_SUPERUSER environment variable is True, it create a superuser.
if [[ $CREATE_SUPERUSER ]];
then
  python manage.py createsuperuser --no-input
fi
```

- Instale os pacotes **Uvicorn** e **Gunicorn** na venv do projeto: `pip install gunicorn uvicorn`.
- Teste se a aplicação esta rodando localmente (substitua **mysite** pelo nome do projeto): `python -m gunicorn mysite.asgi:application -k uvicorn.workers.UvicornWorker`
### Atualize o requirements.txt
`pip freeze > requirements.txt`

## Fazendo o deploy

### Banco de dados
- Crie um novo [banco PostgreSQL](https://render.com/docs/postgresql-creating-connecting) no Render
- Copie o conteúdo em **internal database URL**, ele será usado mais a frente.

### Serviço web
- Crie um novo serviço web no Render
- Sete ele para o seu projeto disponível no GitHub, GitLab, GitBucket e etc.
- Selecione `python 3` como linguagem
- Defina `./build.sh` em **Build Command**
- Defina `python -m gunicorn mysite.asgi:application -k uvicorn.workers.UvicornWorker` em **Start Command** (Mude **mysite** para o nome do projeto)
- Vá em variáveis de ambiente e adicione:
  - `DATABASE_URL`: Cole o conteúdo em **internal database URL**
  - `SECRET_KEY`: Clique no lado para gerar um novo
  - `WEB_CONCURRENCY`: 4

OBS: A conta gratuita não permite usar o shell e com isso não conseguimos criar um superuser, para contornar isso, adicionamos lá no arquivo build.sh a condicional para checar a variável CREATE_SUPERUSER. Vamos adiciona-la nas variáveis de ambiente, bem como os dados desse superuser:
- Ainda em variáveis de ambiente e adicione:
	- `CREATE_SUPERUSER`: True
	- `DJANGO_SUPERUSER_EMAIL`: Informe o email do superuser
	- `DJANGO_SUPERUSER_PASSWORD`: Informe a senha de login do superuser
	- `DJANGO_SUPERUSER_USERNAME`: Informe o username do superuser