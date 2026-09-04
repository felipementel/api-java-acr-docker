# Usuarios API Java

Exemplo do mesmo CRUD de usuarios das pastas `python`, `dotnet` e `node`, agora implementado em Java com Maven e Spring Boot, mantendo Swagger, persistencia em memoria, testes automatizados e arquitetura Ports and Adapters.

## Tecnologias

- Java 25+
- Maven Wrapper
- Spring Boot 4.1.1
- Spring Web MVC
- Spring Validation
- Springdoc OpenAPI
- JUnit 5

## Pre-requisitos

- Windows com PowerShell
- Java 25 ou superior instalado e configurado no `PATH`
- Maven Wrapper incluido no projeto (`mvnw` e `mvnw.cmd`)

Para validar a instalacao:

```powershell
java -version
.\mvnw.cmd -version
```

Se um desses comandos nao funcionar, primeiro ajuste o `JAVA_HOME` e o `PATH` da maquina.

Se o `mvn -version` mostrar uma versao anterior ao Java 25, o build vai falhar com `release version 25 not supported`. O projeto compila normalmente com Java 25 ou superior.

Exemplo de ajuste so para a sessao atual do PowerShell:

```powershell
$env:JAVA_HOME = 'C:\Users\felipe.augusto\.jdks\ms-25.0.1'
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"
.\mvnw.cmd -version
```

Se o seu `~/.m2/settings.xml` corporativo redirecionar tudo com `mirrorOf=*`, projetos pessoais podem falhar com `401 Unauthorized` antes mesmo de baixar o Spring Boot. Nesse caso, use um arquivo de settings separado para este projeto.

## Settings Maven corporativo x pessoal

Voce pode manter o seu `~/.m2/settings.xml` do trabalho como esta para os projetos corporativos. O ponto importante e este:

- `profile` nao resolve esse caso sozinho quando o problema esta em `mirrors`
- `mirrors` no `settings.xml` sao globais
- com `mirrorOf=*`, o Maven redireciona qualquer repositorio para o Nexus corporativo
- por isso, mesmo adicionando um profile para Maven Central, o mirror corporativo continua interceptando a resolucao

Para este projeto, a abordagem mais segura e usar um settings separado no comando Maven.

Existe um exemplo em `settings-public.xml.example`. Para usar localmente:

```powershell
Copy-Item .\settings-public.xml.example .\settings-public.xml
.\mvnw.cmd -s .\settings-public.xml spring-boot:run
```

O mesmo vale para testes e build:

```powershell
.\mvnw.cmd -s .\settings-public.xml test
.\mvnw.cmd -s .\settings-public.xml clean package
```

Se quiser, voce pode ate manter dois modos de uso no mesmo computador:

- corporativo em outros projetos: `mvn ...`
- pessoal neste projeto: `.\mvnw.cmd -s .\settings-public.xml ...`

## Arquitetura

Estrutura do projeto:

```text
src/main/java/com/modelos/usuarios
|-- adapters
|   |-- inbound/http
|   `-- outbound/repositories
|-- application
|   `-- services
|-- domain
|   `-- ports
`-- UsuariosApiApplication.java

src/test/java/com/modelos/usuarios
|-- api
`-- application
```

Responsabilidades:

- `domain`: entidade, erros e porta do repositorio
- `application`: comando de entrada e servico de negocio
- `adapters/outbound`: persistencia em memoria
- `adapters/inbound/http`: controllers e contratos HTTP
- `UsuariosApiApplication`: bootstrap da aplicacao Spring Boot

## Modelo de dominio

- `id: int`
- `nome: string`
- `dtNascimento: date`
- `status: bool`
- `telefones: string[]`

## Como rodar

Na pasta `java`:

### 1. Rodar os testes

```powershell
.\mvnw.cmd -s .\settings-public.xml test
```

### 2. Subir a API

```powershell
.\mvnw.cmd -s .\settings-public.xml spring-boot:run
```

### 3. Acessar a aplicacao

- API: `http://localhost:8080`
- Swagger UI: `http://localhost:8080/docs`
- OpenAPI JSON: `http://localhost:8080/openapi`

### 4. Build empacotado

```powershell
.\mvnw.cmd -s .\settings-public.xml clean package
java -jar .\target\usuarios-api-java-0.1.0.jar
```

## Como testar

```powershell
.\mvnw.cmd -s .\settings-public.xml test
```

## Pre-checks antes do commit

Na raiz do repositorio, execute a verificacao de vulnerabilidades com Trivy:

```powershell
trivy fs --scanners vuln --format table --dependency-tree .
```

O CI tambem executa `./mvnw -B verify`.

## Endpoints

- `GET /health/live`
- `GET /health/ready`
- `POST /usuarios`
- `GET /usuarios`
- `GET /usuarios/{usuarioId}`
- `PUT /usuarios/{usuarioId}`
- `DELETE /usuarios/{usuarioId}`

## Exemplo de payload

```json
{
	"id": 1,
	"nome": "Carlos",
	"dtNascimento": "1992-03-14",
	"status": true,
	"telefones": [
		"11911112222",
		"1122223333"
	]
}
```

## CI

O workflow fica em `.github/workflows/ci.yml` e executa:

- setup do Java 25
- cache Maven
- `./mvnw -B verify`

## Observacoes

- A persistencia e totalmente em memoria.
- Ao reiniciar a aplicacao, os dados sao perdidos.
- O endpoint `/health/ready` retorna `503` se o servico principal nao estiver disponivel.

## Comandos uteis

```powershell
.\mvnw.cmd -s .\settings-public.xml test
.\mvnw.cmd -s .\settings-public.xml spring-boot:run
.\mvnw.cmd -s .\settings-public.xml clean package
```
