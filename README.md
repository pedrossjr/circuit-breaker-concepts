https://marcdias.com.br/vamos-implementar-o-circuit-breaker-pattern/

https://medium.com/trainingcenter/design-pattern-para-microservices-circuit-breaker-f4a5b68f73d1

## Fundamentos básico sobre Circuit Breaker

### O que é Circuit Breaker?

Em uma situação real, pense em um disjuntor elétrico na sua casa. Se houver uma sobrecarga (como muitos aparelhos ligados), o disjuntor "abre" (desliga) para evitar um incêndio ou dano maior. Depois de um tempo, você pode "fechar" ele novamente para testar se o problema foi resolvido.  

No software: Em aplicações onde um serviço chama outro (ex: um app de e-commerce chamando um serviço de pagamento), se o serviço chamado falhar repetidamente (por lentidão, erro ou sobrecarga), o Circuit Breaker "abre" para impedir que as chamadas continuem falhando e consumam recursos desnecessariamente. Isso previne um efeito cascata de falhas no sistema inteiro.  

Por que usar? Em ambientes distribuídos, falhas são inevitáveis (rede cai, servidor sobrecarregado). O Circuit Breaker ajuda a:
- Reduzir latência (não fica esperando respostas que não vêm).
- Evitar sobrecarga em serviços falhando.
- Permitir recuperação automática.

### Estados de um Circuit Breaker

**Closed (Fechado):** Estado normal. Todas as chamadas passam. Ele monitora falhas (ex: contagem de erros ou timeouts).  

**Open (Aberto):** Se o número de falhas ultrapassar um limite (threshold), ele "abre". Nenhuma chamada real é feita; em vez disso, retorna um erro imediato ou um fallback (resposta alternativa, como "Tente mais tarde").  

**Half-Open (Meio-Aberto):** Após um tempo de cooldown (ex: 30 segundos), ele permite uma ou poucas chamadas de teste. Se der certo, volta para Closed; se falhar, volta para Open.

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

*Está é uma simulação simples de um Circuit Breaker para entendimento do seu fundamento. Em ambientes de produção, você pode utilizar o Resilience4j para Java com Spring Boot ou o Hystrix para versões mais antigas como por exemplo Java 11. Já com o .NET você pode utilizar o Polly.*

Hystrix: Este é de longe o mais famoso de todos. É uma biblioteca Java criada pelo Netflix. O Hystrix também possui um dashboard próprio para monitorar os serviços.
PyBreaker: Como o nome já entrega, esta é uma biblioteca do Python. É uma das mais famosas — de acordo com o git stars ⭐️— da linguagem.
Polly: O Polly é uma biblioteca que garante a resiliência de aplicações .NET. Ela implementa diversos algoritmos para garantir isso, um deles é o Circuit Breaker.
Opossum: Uma das bibliotecas Circuit Breakers para Node. (Na verdade, existem diversas opções, antes de escolher a opossum testei a brakes e a levee, das três achei esta mais simples 😃).



### Possíveis erros em ambientes de produção e possíveis soluções

Em ambientes de produção, o Circuit Breakers são usados em sistemas como Netflix, AWS ou apps de grande escala mas, erros podem ocorrer. Citarei alguns abaixo:

- **Thresholds mal configurados**

**Descrição:** Se o limite de falhas for baixo demais por exemplo, abrindo com 1 falha, o circuito abrirá por flutuações normais como picos de rede, causando falsos positivos e indisponibilidade desnecessária. Se for configurado alto demais, demora para detectar falhas reais, permitindo erros em cascata.

**Solução:** Monitore métricas reais utilizando ferramentas como Prometheus ou Datadog. Realize o ajuste baseado em dados começando com valores conservadores por exemplo com 5 falhas em 10 segundos e teste com load testing por exemplo com o [JMeter](https://jmeter.apache.org/) um software de código aberto, uma aplicação 100% Java, projetada para realizar testes de carga, comportamento funcional e medir desempenho de aplicações web.

- **Não lidar com timeouts corretamente**

**Descrição:** Se o serviço chamado demora (slow response), mas não é contado como falha, o sistema trava esperando. Em ambientes de produção, isso acontece em APIs de terceiros sobrecarregadas.

**Solução:** Incluir timeouts na lógica. Use fallbacks retornando dados em cache ou uma mensagem amigável. Em bibliotecas, configure "timeout threshold".

- **Falta de monitoramento ou logging**

**Descrição:** O circuito abre, mas ninguém sabe por quê. Em ambientes de produção, isso leva a downtime prolongado sem alertas.

**Solução:** Integre com sistemas de observabilidade por exemplo como o [ELK Stack](https://www.elastic.co/elastic-stack) para logs, enviando alertas (Slack, PagerDuty) quando o estado mudar, registrando os motivos de falhas para análise post-mortem.

- **Não testar recuperação (Half-Open)**

**Descrição:** No Half-Open, se muitas requisições teste falharem, pode sobrecarregar o serviço recuperando por exemplo em um cluster, onde todos os nodes testam ao mesmo tempo o serviço.

**Solução:** Limite chamadas no Half-Open por exemplo com 1 só por vez. Use jitter (atraso randômico) para evitar thundering herd (avalanche de requests).

- **Ignorar contextos diferentes**

**Descrição:** Um Circuit Breaker global para todos os usuários pode abrir para todos se um grupo causa falhas (ex: ataque DDoS localizado).

**Solução:** Use Circuit Breakers por usuário, região ou tipo de request (per-instance breakers). Em microservices, aplique por endpoint.


Comece praticando com esse MVP no seu código local. Se quiser aprofundar, leia sobre microservices no livro "Building Microservices" de Sam Newman. Qualquer dúvida, pergunta! 😊
