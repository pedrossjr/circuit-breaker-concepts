## Circuit Breaker um design pattern para microserviços

### O que é Circuit Breaker?

A tradução literária para o nome Circuit Breaker é “Disjuntor” então, fazendo uma analogia ao dispositivo de segurança elétrica de uma residência, quando há uma sobrecarga de energia devido a muitos aparelhos ligados, o disjuntor “abre”, desligando a energia para não ocorrer danos aos aparelhos ou um problema maior como um incêndio. 

Na engenharia de software o Circuit Breaker é um design pattern utilizado em microserviços, para garantir a proteção da aplicação de receber novas requisições assim que for detectado sucessivas falhas, impedindo que estas continuem a consumir recursos desnecessários do serviço de aplicação.

Em ambientes de produção, as falhas são inevitáveis e pode ter certeza, tudo é uma questão de tempo, o servidor sobrecarregará, a rede cairá e é neste momento que o Circuit Breaker entra em ação através de algumas regras definidas. Ele monitora a aplicação e quando falhas recorrentes são identificadas o disjuntor “abre” para evitar novas falhas e mensagens não amigáveis para o cliente.

Quando o Circuit Breaker está aberto, ele pode retornar uma resposta alternativa (fallback), um erro imediato informando ao cliente que o serviço está temporariamente indisponível ou dados em cache. Após um período de tempo, o Circuit Breaker pode entrar em um estado de teste (half-open) para verificar se o serviço se recuperou, permitindo algumas requisições de teste. Se as requisições de teste forem bem-sucedidas, o Circuit Breaker fecha novamente e permite que as requisições normais sejam processadas.

O Circuit Breaker em geral ajuda a:

- Reduzir latência de espera por respostas que não serão retornadas.
- Evitar sobrecarga ao serviço, devida as falhas recorrentes de requisições.
- Recuperação automática, quando não são mais detectados problemas.

### Estados de um Circuit Breaker

**Closed (Fechado):** Estado normal. Todas as chamadas passam. Ele monitora falhas (ex: contagem de erros ou timeouts).

**Open (Aberto):** Se o número de falhas ultrapassar um limite (threshold), ele "abre". Nenhuma chamada real é feita; em vez disso, retorna um erro imediato ou um fallback (resposta alternativa, como "Não foi possível processar sua requisição. Tente novamente mais tarde").

**Half-Open (Meio-Aberto):** Após um tempo de cooldown (ex: 30 segundos), ele permite uma ou poucas chamadas de teste. Se der certo, volta para o estado __Closed__; se falhar, volta para __Open__.

<div align="center">
<img src="https://github.com/pedrossjr/circuit-breaker-concepts/blob/main/img/circuit-breaker.png">
</div>  

Esta abordagem é baseada no livro ["Release It!: Design and Deploy Production-Ready Software (Pragmatic Programmers) 1st Edition" de Michael T. Nygard](https://www.amazon.com/Release-Production-Ready-Software-Pragmatic-Programmers/dp/0978739213), que popularizou este padrão.

### Simulação simples em Python de um Circuit Breaker  

Abaixo segue um exemplo de código em Python para entendimento deste conceito.  

```
import time
import random

class CircuitBreaker:
    def __init__(self, failure_threshold=3, timeout=5, retry_timeout=10):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.retry_timeout = retry_timeout
        self.state = "CLOSED"
        self.failure_count = 0
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.retry_timeout:
                self.state = "HALF-OPEN"
            else:
                raise Exception("Circuit is OPEN - Try again later.")

        try:
            result = func(*args, **kwargs)
            self._success()
            return result
        except Exception as e:
            self._failure()
            raise e

    def _success(self):
        self.failure_count = 0
        if self.state == "HALF-OPEN":
            self.state = "CLOSED"

    def _failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold and self.state != "OPEN":
            self.state = "OPEN"
        elif self.state == "HALF-OPEN":
            self.state = "OPEN"

def unreliable_service():
    if random.random() > 0.5:
        raise Exception("Service failed!")
    return "Success!"

cb = CircuitBreaker(failure_threshold=2, retry_timeout=5)

for i in range(10):
    try:
        result = cb.call(unreliable_service)
        print(f"Attempt {i+1}: {result} (State: {cb.state})")
    except Exception as e:
        print(f"Attempt {i+1}: {e} (State: {cb.state})")
    time.sleep(1)
```

Explicação do código:

**Classe CircuitBreaker:** Gerencia os estados e contagens.

**call():** Método que "protege" a chamada à função. Se Open, bloqueia; se Half-Open, testa.  

**unreliable_service():** Função que falha aleatoriamente para simular um serviço real.

**Execução:** Execute o código Python, após 2 falhas o circuit breaker será aberto, esperando 5 segundos, e após tentando recuperar.

Está foi uma simulação simples de um Circuit Breaker para entendimento do seu fundamento para uso em ambientes de produção atualmente você pode utilizar o [**Resilience4j**](https://resilience4j.readme.io/docs/getting-started-3) para Java com Spring Boot ou o [**Hystrix**](https://github-com.translate.goog/Netflix/Hystrix?_x_tr_sl=en&_x_tr_tl=pt&_x_tr_hl=pt&_x_tr_pto=tc) para versões mais antigas. Existem também o [**PyBreaker**](https://pypi.org/project/pybreaker/) que é biblioteca para utilização com Python; o [**Polly**](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/implement-circuit-breaker-pattern) para aplicações .NET; e o Opossum é uma das bibliotecas disponíveis para Node. 

### Possíveis erros em ambientes de produção e possíveis soluções

Em ambientes de produção, o Circuit Breakers são usados em sistemas como Netflix, AWS ou apps de grande escala mas, erros podem ocorrer. Citarei alguns abaixo:

**Thresholds mal configurados**

**Descrição:** Se o limite de falhas for baixo demais por exemplo, abrindo com 1 falha, o circuito abrirá por flutuações normais como picos de rede, causando falsos positivos e indisponibilidade desnecessária. Se for configurado alto demais, demora para detectar falhas reais, permitindo erros em cascata.

**Solução:** Monitore métricas reais utilizando ferramentas como Prometheus ou Datadog. Realize o ajuste baseado em dados começando com valores conservadores por exemplo com 5 falhas em 10 segundos e teste com load testing por exemplo com o [JMeter](https://jmeter.apache.org/) um software de código aberto projetado para realizar testes de carga, comportamento funcional e medir desempenho de aplicações web.

**Não lidar com timeouts corretamente**

**Descrição:** Se o serviço chamado demora (slow response), mas não é contado como falha, o sistema trava esperando. Em ambientes de produção, isso acontece em APIs de terceiros sobrecarregadas.

**Solução:** Incluir timeouts na lógica, usar fallbacks para retornar dados em cache ou uma mensagem amigável. Em bibliotecas, configure "timeout threshold".

**Falta de monitoramento ou logging**

**Descrição:** O circuito abre, mas ninguém sabe por quê. Em ambientes de produção, isso leva a downtime prolongado sem alertas.

**Solução:** Integre com sistemas de observabilidade por exemplo como o [ELK Stack](https://www.elastic.co/elastic-stack) para logs, enviando alertas (Slack, PagerDuty) quando o estado mudar, registrando os motivos de falhas para análise post-mortem.

**Não testar recuperação (Half-Open)**

**Descrição:** No Half-Open, se muitas requisições teste falharem, pode sobrecarregar o serviço recuperando em um cluster, onde todos os nodes testam ao mesmo tempo o serviço.

**Solução:** Limite chamadas no Half-Open por exemplo com 1 só por vez. Use jitter (atraso randômico) para evitar thundering herd (avalanche de requests).

**Ignorar contextos diferentes**

**Descrição:** Um Circuit Breaker global para todos os usuários pode abrir para todos se um grupo causa falhas por exeomplo em ataque de DDoS localizado.

**Solução:** Use Circuit Breakers por usuário, região ou tipo de request (per-instance breakers) e emm microserviços aplique por endpoint.

Este pequeno artigo serviu para compartilhar conhecimento e mostrar o que é um Circuit Breaker, para que ele serve e como pode ser utilizado. Para uma melhor experiência, utilize ferramentas de observabilidade como Prometheus e Grafana para monitorar em tempo real suas aplicações.

Espero que tenham gostado! :blush:
