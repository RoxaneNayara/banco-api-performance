# Banco API — Testes de Performance

## Introdução

Projeto de testes de performance de uma API bancária, desenvolvido com JavaScript e Grafana k6 para fins de estudo e prática. Os cenários cobrem autenticação e transferências, com validações das respostas HTTP e acompanhamento das métricas de execução.

A API deve estar disponível separadamente. Este repositório contém os scripts de teste e seus arquivos de apoio.

## Tecnologias utilizadas

| Tecnologia | Utilização |
| --- | --- |
| JavaScript | Implementação dos cenários e funções auxiliares. |
| Grafana k6 | Execução dos testes, simulação de usuários virtuais, checks e métricas de desempenho. |
| JSON | Armazenamento das configurações locais e dos dados de teste. |
| Git e GitHub | Versionamento e hospedagem do código. |

Os scripts são executados pelo runtime do k6. Não é necessário instalar Node.js ou executar `npm install` para rodar os testes deste repositório.

## Estrutura do repositório

```text
banco-api-performance/
├── config/
│   └── config.local.json
├── fixtures/
│   └── postLogin.json
├── helpers/
│   └── autenticacao.js
├── tests/
│   ├── login.test.js
│   └── transferencias.test.js
├── utils/
│   └── variaveis.js
├── .gitignore
└── README.md
```

## Objetivo de cada grupo de arquivos

| Grupo / arquivo | Objetivo |
| --- | --- |
| `config/config.local.json` | Define a URL base local usada quando `BASE_URL` não é informada. O valor atual é `http://localhost:3000`. |
| `fixtures/postLogin.json` | Armazena o payload utilizado na autenticação. Deve conter dados válidos para a API de teste. |
| `helpers/autenticacao.js` | Disponibiliza `obterToken()`, que faz uma requisição a `/login` e retorna o campo `token` da resposta. |
| `tests/login.test.js` | Executa requisições a `/login`, valida status HTTP 200 e token do tipo string, e aplica um perfil de carga com limites de desempenho. |
| `tests/transferencias.test.js` | Obtém um token, envia uma transferência para `/transferencias` com autenticação Bearer e valida status HTTP 201. Atualmente executa uma iteração. |
| `utils/variaveis.js` | Centraliza a obtenção da URL base por meio de `pegarBaseURL()`, priorizando `__ENV.BASE_URL` sobre a configuração local. |
| `.gitignore` | Define os arquivos e diretórios ignorados pelo Git. |
| `README.md` | Documenta a organização, preparação e execução do projeto. |

### Perfil atual do teste de login

| Etapa | Duração | Usuários virtuais ao final da etapa |
| --- | --- | --- |
| Aumento gradual | 5 segundos | 10 |
| Carga constante | 20 segundos | 10 |
| Redução gradual | 5 segundos | 0 |

Limites configurados em `thresholds`:

- `http_req_duration: p(90)<3000`: o percentil 90 do tempo das requisições deve ser inferior a 3.000 ms.
- `http_req_duration: max<5000`: o maior tempo de requisição deve ser inferior a 5.000 ms.
- `http_req_failed: rate<0.01`: a taxa de falha HTTP deve ser inferior a 1%.

Os checks verificam condições nas respostas. Os thresholds definem os critérios de desempenho que determinam a aprovação da execução. Atualmente, o script de transferências possui um check de status, mas não define thresholds de desempenho.

## Instalação e preparação

### 1. Pré-requisitos

- Git instalado.
- Grafana k6 instalado, em uma versão com suporte ao web dashboard integrado.
- API bancária em execução e acessível.
- Usuário válido para autenticação e contas aptas a realizar a transferência de teste.

### 2. Instalar o k6

**macOS, com Homebrew:**

```bash
brew install k6
```

**Windows, com Winget:**

```powershell
winget install k6 --source winget
```

Para Linux e outras formas de instalação, consulte o [guia oficial de instalação do k6](https://grafana.com/docs/k6/latest/set-up/install-k6/).

Verifique a instalação:

```bash
k6 version
```

### 3. Clonar o repositório

```bash
git clone https://github.com/RoxaneNayara/banco-api-performance.git
cd banco-api-performance
```

### 4. Preparar a API e os dados

Inicie a API conforme as instruções do projeto que a fornece. A inicialização da API não é feita por este repositório.

Confira os dados em `fixtures/postLogin.json`. No teste de login, o campo `username` é sobrescrito pelo valor `julio.lima`; o helper de autenticação utiliza os dados do arquivo diretamente.

No teste de transferências, os dados atuais são:

```json
{
  "contaOrigem": 1,
  "contaDestino": 2,
  "valor": 11,
  "token": ""
}
```

A autenticação é enviada no cabeçalho `Authorization: Bearer <token>`. O campo `token` do payload permanece vazio no script atual.

Confirme que as contas existem e que a conta de origem possui saldo suficiente. A execução realiza transferências e pode alterar os dados do ambiente.

## Execução do projeto

Execute os comandos a partir da raiz do repositório.

### Informar a URL base com BASE_URL

Passe `BASE_URL` usando a opção `-e` do k6. Informe o endereço base da API, sem a barra final e sem os caminhos `/login` ou `/transferencias`:

```bash
k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

A variável é lida em `utils/variaveis.js`:

```javascript
return __ENV.BASE_URL || configLocal.baseUrl;
```

Embora exista um fallback local, prefira informar `BASE_URL` explicitamente em cada execução para identificar o ambiente utilizado.

### Executar o teste de login

```bash
k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

O teste utiliza as etapas de carga e os thresholds definidos em `tests/login.test.js`.

### Executar o teste de transferências

```bash
k6 run -e BASE_URL=http://localhost:3000 tests/transferencias.test.js
```

A configuração atual realiza uma iteração, incluindo a autenticação e a transferência. Ela permite verificar o fluxo, mas não representa um teste de carga sustentada.

### Acompanhar os resultados em tempo real

No macOS ou Linux (Bash/Zsh), habilite o dashboard com uma variável de ambiente do k6:

```bash
K6_WEB_DASHBOARD=true k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

Durante a execução, abra [http://localhost:5665](http://localhost:5665) no navegador.

Para abrir o dashboard automaticamente:

```bash
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_OPEN=true k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

### Acompanhar e exportar o relatório HTML

No macOS ou Linux (Bash/Zsh):

```bash
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=html-report.html k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

Esse comando habilita o acompanhamento em tempo real e gera `html-report.html` no diretório de execução ao final do teste. O arquivo pode ser aberto no navegador e compartilhado para consulta dos resultados.

Para executar o teste de transferências com exportação:

```bash
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=transferencias-report.html k6 run -e BASE_URL=http://localhost:3000 tests/transferencias.test.js
```

**Windows (PowerShell):**

```powershell
$env:K6_WEB_DASHBOARD = "true"
$env:K6_WEB_DASHBOARD_EXPORT = "html-report.html"
k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
```

No PowerShell, essas variáveis permanecem definidas na sessão. Para removê-las após o uso:

```powershell
Remove-Item Env:K6_WEB_DASHBOARD
Remove-Item Env:K6_WEB_DASHBOARD_EXPORT
```

| Variável | Finalidade |
| --- | --- |
| `BASE_URL` | Define o endereço base da API utilizado pelos scripts. |
| `K6_WEB_DASHBOARD` | Habilita o dashboard integrado quando definida como `true`. |
| `K6_WEB_DASHBOARD_OPEN` | Abre o dashboard automaticamente no navegador. |
| `K6_WEB_DASHBOARD_EXPORT` | Define o caminho do relatório HTML exportado ao final da execução. |
| `K6_WEB_DASHBOARD_PERIOD` | Define o intervalo de agregação e atualização do dashboard; o padrão é `10s`. |

`BASE_URL` é uma variável do projeto e pode ser passada com `-e`. As variáveis `K6_WEB_DASHBOARD*` configuram o k6 e devem ser definidas no ambiente do processo, conforme os exemplos acima.

**Observações sobre o relatório:**

- Para incluir gráficos, a duração do teste deve ser maior que três vezes o período de agregação do dashboard. Com o padrão de `10s`, ela precisa ser maior que 30 segundos. O teste de login está configurado para 30 segundos, portanto não garante gráficos no HTML com esse período.
- Para o teste de login atual, é possível usar um período de `1s`:

  ```bash
  K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_PERIOD=1s K6_WEB_DASHBOARD_EXPORT=html-report.html k6 run -e BASE_URL=http://localhost:3000 tests/login.test.js
  ```

- O teste de transferências, por executar apenas uma iteração, pode terminar antes de produzir dados suficientes para os gráficos.
- Se o processo continuar aberto após o teste, feche a aba do dashboard: o k6 pode aguardar enquanto houver uma conexão ativa com ele.
- Use nomes diferentes para preservar os relatórios de cada execução.

Referências: [Web dashboard e exportação HTML](https://grafana.com/docs/k6/latest/results-output/web-dashboard/) · [Variáveis de ambiente no k6](https://grafana.com/docs/k6/latest/using-k6/environment-variables/).
