https://marcdias.com.br/vamos-implementar-o-circuit-breaker-pattern/

https://medium.com/trainingcenter/design-pattern-para-microservices-circuit-breaker-f4a5b68f73d1

## Circuit Breaker um design pattern para uso com microserviços

### O que é Circuit Breaker?

O Circuit Breaker é a tradução literária para “Disjuntor” então, fazendo uma analogia a engenharia elétrica, pense em um disjuntor de energia em uma residência, quando há uma sobrecarga de energia, devido a muitos aparelhos ligados, o disjuntor, também conhecido com DR, “abre” ou seja, desliga a energia para que não ocorra danos aos aparelhos ou algo maior como um incêndio por exemplo. Após a correção da elétrica, o eletricista pode “fechar“ o disjuntor novamente para testar se o problema foi resolvido, caso o contrário ele o mantém aberto até tudo estar resolvido.

Na engenharia de software o Circuit Breaker funciona da mesma maneira. É um Design Pattern muito utilizado em microserviços, garantindo a proteção da aplicação de receber novas requisições assim que for detectado um problema que cause repetidas falhas de requisição impedindo que estas requisições continuem, falhando sucessivamente e consumindo recursos desnecessários do servidor de aplicação. Esta ação previne contra efeito em cascata de falhas no sistema por inteiro.

Em sistemas distribuídos em ambientes de produção, as falhas são inevitáveis e tudo é uma questão de tempo, o servidor sobrecarrega, a rede cai e neste momento que o Circuit Breaker entra em ação, através de algumas regras definidas. Ele monitora a aplicação e quando falhas recorrentes são identificadas o disjuntor “abre” para evitar mais problemas impedindo posteriormente uma catástrofe sistêmica.

O Circuit Breaker em geral ajuda a:

- Reduzir latência (espera por respostas que não virão).
- Evitar sobrecarga (falhas recorrentes de requisições causando sobrecarga no serviço).
- Recuperação automática (Automaticamente o serviço retorna quando não são mais detectados problemas).

### Estados de um Circuit Breaker

**Closed (Fechado):** Estado normal. Todas as chamadas passam. Ele monitora falhas (ex: contagem de erros ou timeouts).

**Open (Aberto):** Se o número de falhas ultrapassar um limite (threshold), ele "abre". Nenhuma chamada real é feita; em vez disso, retorna um erro imediato ou um fallback (resposta alternativa, como "Não foi possível processar sua requisição. Tente novamente mais tarde").

**Half-Open (Meio-Aberto):** Após um tempo de cooldown (ex: 30 segundos), ele permite uma ou poucas chamadas de teste. Se der certo, volta para o estado __Closed__; se falhar, volta para __Open__.

Esta abordagem é baseada no livro ["Release It!: Design and Deploy Production-Ready Software (Pragmatic Programmers) 1st Edition" de Michael T. Nygard](https://www.amazon.com/Release-Production-Ready-Software-Pragmatic-Programmers/dp/0978739213), que popularizou este padrão.  

### Simulação simples de um Circuit Breaker  

Abaixo seguem um exemplo de código em Python para entendimento deste conceito.  

Imagine o seguinte cenário onde, um serviço que chama uma API externa que às vezes falha e o Circuit Breaker protege este serviço contra falhas repetidas.  

**Código em Python**

```
import time
import random

# Classe CircuitBreaker
class CircuitBreaker:
    # Método construtor da classe
    def __init__(self, failure_threshold=3, timeout=5, retry_timeout=10):
        self.failure_threshold = failure_threshold # Limite de falhas para abrir
        self.timeout = timeout # Tempo de espera por resposta
        self.retry_timeout = retry_timeout # Tempo de cooldown no estado Open
        self.state = "FECHADO"
        self.failure_count = 0
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == "ABERTO":
            # Verifica se é hora de tentar Half-Open
            if time.time() - self.last_failure_time > self.retry_timeout:
                self.state = "MEIO-ABERTO"
            else:
                raise Exception("O Circuit Breaker está ABERTO - Tente novamente mais tarde.")

        try:
            result = func(*args, **kwargs) # Executa a função
            self._success() # Sucesso: reseta contadores
            return result
        except Exception as e:
            self._failure() # Falha: conta e possivelmente abre
            raise e

    def _success(self):
        self.failure_count = 0
        if self.state == "MEIO-ABERTO":
            self.state = "FECHADO" # Volta para Fechado se houver sucesso em MEIO-ABERTO (Half-Open)

    def _failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold and self.state != "ABERTO":
            self.state = "ABERTO" # Abre o circuito
        elif self.state == "MEIO-ABERTO":
            self.state = "ABERTO" # Falha no teste: volta para Open

# Função unreliable_service() que pode falhar com 50% de chance, simulando a chamada a um serviço
def unreliable_service():
    if random.random() > 0.5:
        raise Exception("O serviço falhou!")
    return "Chamada com successo!"

# Criando uma instância da classe CircuitBreaker
cb = CircuitBreaker(failure_threshold=2, retry_timeout=5)

for i in range(30):
    try:
        result = cb.call(unreliable_service)
        print(f"Tentativa {i+1}: {result} (Estado: {cb.state})")
    except Exception as e:
        print(f"Tentativa {i+1}: {e} (Estado: {cb.state})")
    time.sleep(1) # Simula o tempo entre chamadas
```

Explicação do código:

**Classe CircuitBreaker:** Gerencia os estados e contagens.

**call():** Método que "protege" a chamada à função. Se Open, bloqueia; se Half-Open, testa.  

**unreliable_service():** Função que falha aleatoriamente para simular um serviço real.

**Execução:** Execute o código Python, após 2 falhas o circuit breaker será aberto, esperando 5 segundos, e após tentando recuperar.

Está foi uma simulação simples de um Circuit Breaker para entendimento do seu fundamento para uso em ambientes de produção atualmente você pode utilizar o **Resilience4j** para Java com Spring Boot ou o Hystrix para versões mais antigas. Existem também o **PyBreaker**  que é biblioteca para utilização com Python; o Polly para aplicações .NET; e o Opossum é uma das bibliotecas disponíveis para Node. 

### Possíveis erros em ambientes de produção e possíveis soluções

Em ambientes de produção, o Circuit Breakers são usados em sistemas como Netflix, AWS ou apps de grande escala mas, erros podem ocorrer. Citarei alguns abaixo:

- **Thresholds mal configurados**

**Descrição:** Se o limite de falhas for baixo demais por exemplo, abrindo com 1 falha, o circuito abrirá por flutuações normais como picos de rede, causando falsos positivos e indisponibilidade desnecessária. Se for configurado alto demais, demora para detectar falhas reais, permitindo erros em cascata.

**Solução:** Monitore métricas reais utilizando ferramentas como Prometheus ou Datadog. Realize o ajuste baseado em dados começando com valores conservadores por exemplo com 5 falhas em 10 segundos e teste com load testing por exemplo com o [JMeter](https://jmeter.apache.org/) um software de código aberto projetado para realizar testes de carga, comportamento funcional e medir desempenho de aplicações web.

- **Não lidar com timeouts corretamente**

**Descrição:** Se o serviço chamado demora (slow response), mas não é contado como falha, o sistema trava esperando. Em ambientes de produção, isso acontece em APIs de terceiros sobrecarregadas.

**Solução:** Incluir timeouts na lógica, usar fallbacks para retornar dados em cache ou uma mensagem amigável. Em bibliotecas, configure "timeout threshold".

- **Falta de monitoramento ou logging**

**Descrição:** O circuito abre, mas ninguém sabe por quê. Em ambientes de produção, isso leva a downtime prolongado sem alertas.

**Solução:** Integre com sistemas de observabilidade por exemplo como o [ELK Stack](https://www.elastic.co/elastic-stack) para logs, enviando alertas (Slack, PagerDuty) quando o estado mudar, registrando os motivos de falhas para análise post-mortem.

- **Não testar recuperação (Half-Open)**

**Descrição:** No Half-Open, se muitas requisições teste falharem, pode sobrecarregar o serviço recuperando em um cluster, onde todos os nodes testam ao mesmo tempo o serviço.

**Solução:** Limite chamadas no Half-Open por exemplo com 1 só por vez. Use jitter (atraso randômico) para evitar thundering herd (avalanche de requests).

- **Ignorar contextos diferentes**

**Descrição:** Um Circuit Breaker global para todos os usuários pode abrir para todos se um grupo causa falhas por exeomplo em ataque de DDoS localizado.

**Solução:** Use Circuit Breakers por usuário, região ou tipo de request (per-instance breakers) e emm microserviços aplique por endpoint.

Este pequeno artigo serviu para compartilhar conhecimento e mostrar o que é um Circuit Breaker, para que ele serve e como pode ser utilizado. Espero que tenham gostado!
