
# Exemplo de Implementação de Circuit Breaker com Spring Boot

Este tutorial demonstra como implementar o padrão **Circuit Breaker** em uma aplicação Java usando **Spring Boot** e a biblioteca **Resilience4j**. 

O Circuit Breaker ajuda a proteger a aplicação contra falhas em cascata, evitando chamadas excessivas para serviços que estão falhando.

Para gerar um projeto Springboot, utilize o Spring Initializr: https://start.spring.io/

Você também precisará ter instalado o Java +16 e Maven +3.9.9

## Passos para Implementar o Circuit Breaker

### 1. Dependências do Maven

Adicione as dependências necessárias no arquivo `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot2</artifactId>
        <version>1.7.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### 2. Configuração no `application.properties`

Adicione a configuração do Circuit Breaker no arquivo de propriedades da sua aplicação.

```properties
# Configurações do Circuit Breaker para a instância "backendA"
resilience4j.circuitbreaker.instances.backendA.slidingWindowSize=10
resilience4j.circuitbreaker.instances.backendA.failureRateThreshold=50
resilience4j.circuitbreaker.instances.backendA.waitDurationInOpenState=5000
resilience4j.circuitbreaker.instances.backendA.permittedNumberOfCallsInHalfOpenState=3
resilience4j.circuitbreaker.instances.backendA.registerHealthIndicator=true

# Porta da aplicação (opcional)
server.port=8080
```

Aqui, `backendA` é o nome da instância do Circuit Breaker, onde você pode ajustar parâmetros como a taxa de falhas (`failureRateThreshold`) e o tempo que o Circuit Breaker permanecerá no estado aberto (`waitDurationInOpenState`).

### 3. Serviço com Circuit Breaker

Crie uma classe de serviço onde o Circuit Breaker será aplicado.

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ExternalService {

    private final RestTemplate restTemplate = new RestTemplate();

    @CircuitBreaker(name = "backendA", fallbackMethod = "fallbackResponse")
    public String callExternalService() {
        // Simulando chamada para um serviço externo
        String url = "https://url-inexistente.com/dados";
        return restTemplate.getForObject(url, String.class);
    }

    // Método fallback que será chamado quando o Circuit Breaker abrir
    public String fallbackResponse(Throwable throwable) {
        return "Serviço indisponível no momento. Tente novamente mais tarde.";
    }
}
```

### 4. REST Controller

Crie um controlador simples que chama o serviço com o Circuit Breaker implementado.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class CircuitBreakerController {

    private final ExternalService externalService;

    public CircuitBreakerController(ExternalService externalService) {
        this.externalService = externalService;
    }

    @GetMapping("/api/dados")
    public String getDados() {
        return externalService.callExternalService();
    }
}
```

### 5. Habilitando o Circuit Breaker

Execute a classe principal da sua aplicação e acesse o endereço `http://localhost:8080/api/dados` no browser.

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Após execução o esperado é que a sua aplicação responda com sucesso, conforme tela abaixo:

<img width="1004" alt="Screen Shot 2024-09-11 at 20 02 16" src="https://github.com/user-attachments/assets/c5995c45-e831-4444-9a97-a3a17903a5b0">

### 6. Simulando Fallback

Para simular o comportamento do fallback, você pode forçar uma exceção no método `callExternalService` ou simular a indisponibilidade de um serviço externo.

##### 6.1 Simulação de Erro com Exceção

```java
@Service
public class ExternalService {

    @CircuitBreaker(name = "backendA", fallbackMethod = "fallbackResponse")
    public String callExternalService() {
        // Simulando uma falha no serviço externo lançando uma exceção
        throw new RuntimeException("Simulando uma falha no serviço externo");
    }

    public String fallbackResponse(Throwable throwable) {
        // Retornando uma resposta fallback
        return "Serviço indisponível no momento. Tente novamente mais tarde.";
    }
}
```

##### 6.2 Simulação de Erro com Serviço Indisponível

```java
@Service
public class ExternalService {

    private final RestTemplate restTemplate = new RestTemplate();

    @CircuitBreaker(name = "backendA", fallbackMethod = "fallbackResponse")
    public String callExternalService() {
        // Simulando uma chamada para um serviço externo que não existe
        String url = "https://url-inexistente.com/dados";
        return restTemplate.getForObject(url, String.class);
    }

    public String fallbackResponse(Throwable throwable) {
        return "Serviço indisponível no momento. Tente novamente mais tarde.";
    }
}
```

### 7. Testando o Fallback

Agora, ao iniciar a aplicação e acessar a rota `/api/dados`, o Circuit Breaker detectará a falha e acionará o método `fallbackResponse`, retornando a mensagem de fallback: **"Serviço indisponível no momento. Tente novamente mais tarde."**

---

Você pode monitorar os logs do Circuit Breaker ativando os logs do Resilience4j no `application.properties`:

```properties
logging.level.io.github.resilience4j.circuitbreaker=DEBUG
```

