# Getting Started

### Reference Documentation

For further reference, please consider the following sections:

* [Official Apache Maven documentation](https://maven.apache.org/guides/index.html)
* [Spring Boot Maven Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/3.0.5/maven-plugin/reference/html/)
* [Create an OCI image](https://docs.spring.io/spring-boot/docs/3.0.5/maven-plugin/reference/html/#build-image)
* [Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#spring-cloud-eureka-server)

### Guides

The following guides illustrate how to use some features concretely:

* [Service Registration and Discovery with Eureka and Spring Cloud](https://spring.io/guides/gs/service-registration-and-discovery/)

взято с сайта:
https://sysout.ru/mikroservisy-eureka-i-client-side-load-balancing/

Микросервисы: Eureka и client-side Load Balancing
В этой статье рассмотрим пример с двумя микросервисами. Обнаруживать друг друга они будут с помощью Eureka. Кроме того, рассмотрим, как запускать микросервисы в нескольких экземплярах и балансировать нагрузку на микросервис (со стороны клиента).

Пример с микросервисами
Итак, пусть у нас имеется два микросервиса (два Spring Boot приложения, предоставляющих REST API). Один — для внешнего пользователя, второй — для того, чтобы к нему обращался первый.

Микросервис Zoo
Сюда внешний пользователь приходит, чтобы посмотреть животных. Он заходит на адрес:

localhost:8081/animals/any
и видит там одно случайно выбранное животное. Например, dog:



Вот контроллер для отображения:

@RestController
public class ZooController {
@Autowired
private RandomAnimalClient randomAnimalClient;
@GetMapping("/animals/any")
ResponseEntity<Animal> seeAnyAnimal(){
return randomAnimalClient.random();
}
}
RandomAnimalClient внедрен потому, что животные на самом деле берутся из второго микросервиса Random Animal. RandomAnimalClient как раз и делает запрос к нему.

Микросервис Random Animal — выдает случайное животное
Второй микросервис — тоже отдельное Spring Boot приложение. Вот его контроллер:

@RestController
public class RandomAnimalController {
private final AnimalDao animalDao;
public NameController(AnimalDao animalDao){
this.animalDao=animalDao;
}
@GetMapping("/random")
public Animal randomAnimal(){
return animalDao.random();
}
}
Этот микросервис запущен в двух экземплярах на портах 8082 и 8083. То есть случайное животное можно получить по любому из адресов:

localhost:8082/random
localhost:8083/random
AnimalDao, внедренный в RandomAnimalController, берет случайное животное из списка:

@Component
public class AnimalDao {
private List<Animal> list = Arrays.asList(new Animal("cat"), new Animal("dog"), new Animal("fox"));
public Animal random(){
Random rand = new Random();
return  list.get(rand.nextInt(list.size()));
}
}
Решение без Spring Cloud и возникающие проблемы
Если решать задачу без использования  замечательных возможностей Spring Cloud, то со временем возникают проблемы:

Придется вести учет url и портов. У нас задача простая — два микросервиса на  трех портах, и это не сложно. Но если микросервисов много, и адреса динамически меняются (запускаются новые экземпляры микросервисов на новых портах, какие-то экземпляры падают)? Хотелось бы, чтобы при запуске и отключении очередного экземпляра микросервиса другие микросервисы были автоматически информированы о появлении и пропаже,  и всё продолжало бы работать без исправления кода.
Надо как-то выбирать, на какой из запущенных экземпляров микросервиса обратиться. Мы запускаем два экземпляра Random Animal. И первый микросервис должен долбить не один и тот же экземпляр, а выбирать их (примерно) по очереди.
Есть еще проблемы, но о них в следующих статьях, а пока про первые две.

Первая проблема решается с помощью сервера Eureka. Мы запускаем отдельное приложение Eureka, которое ведет учет микросервисов и их адресов. Eureka по умолчанию запускается на порту 8761 — клиенты-микросервисы знают номер и уведомляют о себе при старте. Также сама Eureka периодически проверяет, жив ли клиент-микросервис. Чтобы сделать приложение клиентом сервера Eureka, мы добавляем в него Maven-зависимость, и всё. После этого он зарегистрируется в Eureka при запуске автоматически.
Нагрузка балансируется автоматически, если для обращения к экземплярам использовать не просто RestTemplate, а @LoadBalanced RestTemplate. (Можно еще использовать WebClient — он поддерживает реактивность — но о нем не в этой статье).
Решение со Spring Cloud
Итак, сначала о сервере Eureka, которая ведет реестр микросервисов.

Обнаружение сервисов с помощью Eureka Server
Чтобы создать приложение Eureka Server, в POM-файл нужно добавить зависимость:

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
А главный класс нужно аннотировать @EnableEurekaServer:

@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
public static void main(String[] args) {
SpringApplication.run(EurekaApplication.class, args);
}
}
Eureka Server готов. Осталось добавить настройки.

В application.properties пропишем:

eureka.client.registerWithEureka=false
eureka.client.fetchRegistry=false
spring.application.name=eureka-server
spring.cloud.loadbalancer.ribbon.enabled=false
Теоретически сервер может выступать и клиентом. Первые две настройки отменяют эту возможность: не дают серверу регистрировать самого себя в качестве клиента.

Третья настройка задает имя микросервиса, а четвертая отменяет использование Ribbon в качестве балансировщика нагрузки по умолчанию. Эти настройки будут в всех микросервисов. Но о балансировке ниже.

Пока займемся регистрацией микросервисов в качестве клиентов Eureka.

Сами микросервисы — Eureka Clients
Чтобы сделать микросервисы Zoo и Random Animal клиентами Эврики, в них необходимо добавить зависимости:

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
Для каждого микросервиса необходимо задать имя, порт и отключить балансировщик нагрузки Ribbon (потому что будем использовать другой):

spring.application.name=zoo
spring.cloud.loadbalancer.ribbon.enabled=false
server.port=8081
По заданному имени микросервиса они смогут обращаться друг к другу
Все вместе — запуск сервера Eureka и его клиентов
Далее нужно запустить сначала сервер Eureka, а затем клиенты-микросервисы.

Eureka по умолчанию за пускается на порту 8761, и если открыть

http://localhost:8761/
то увидим список запущенных микросервисов с их именами:

Eureka показывает своих клиентов
Eureka показывает своих клиентов zoo и random-animal (2 экземпляра)
Random-animal запущен дважды — на портах 8082 и 8083.

Как запустить экземпляр на другом порту в IDEA
Чтобы указать порт для второго экземпляра Random-animal, в IntelliJ IDEA в Run->Edit Configurations в поле VM Options прописываем:

-Dserver.port=8083
Запуск на конкретном порту
Запуск на конкретном порту
Эта настройка перезаписывает порт, заданный в application.properties.

Балансировка нагрузки с помощью @Loadbalanced RestTemplate
Для обращения одного микросервиса к другому нам понадобятся имена, заданные в настройках (spring.application.name).

Итак, из Zoo к Random Animal мы обращались с помощью RandomAnimalClient, вот его код:

@Component
public class RandomAnimalClient {
private static final Logger LOGGER = LoggerFactory
.getLogger(RandomAnimalClient.class);
private final RestTemplate restTemplate;
RandomAnimalClient(RestTemplate restTemplate) {
this.restTemplate = restTemplate;
}
//spring.application.name=random-animal есть в настройках Random Animal
public ResponseEntity<Animal> random() {
LOGGER.debug("Sending  request for animal {}");
return restTemplate.getForEntity("http://random-animal/random",
Animal.class);
}

}
Но RestTemplate тут не простой, а сбалансированный. Именно поэтому обращение

http://random-animal/random
идет на соседний микросервис с прописанным в настройках именем random-animal, а не в интернет.

Чтобы сделать RestTemplate сбалансированным, просто аннотируем его @LoadBalanced:

@Configuration
public class RestTemplateConfig {
@Bean
@LoadBalanced
RestTemplate restTemplate() {
return new RestTemplate();
}
}
Для использования @LoadBalanced RestTemplate в POM никакой зависимости добавлять не нужно.

Давайте добавим в RandomAnimalController  вывод животного в консоль. Снова всё запустим и будем обновлять главную страницу. В консолях двух запущенных экземпляров Random Animal будет по очереди выводиться животное — то в одной консоли, то в другой. Видно, что балансировка нагрузки происходит:

Консоли экземпляров Random Animal
Консоли экземпляров Random Animal
Рассмотренная балансировка называется client-side load balancing, потому что именно клиент (тот, кто обращается к REST API) решает, к какому именно экземпляру сервиса обратиться. И делает он это с помощью @Loadbalanced RestTemplate.

DiscoveryClient — альтернатива @Loadbalanced RestTemplate
Кстати, код с @Loadbalanced RestTemplate можно переписать на более понятный. Можно взять обычный RestTemplate, внедрить DiscoveryClient в RandomAnimalClient и переписать метод random() так:

@Component
public class RandomAnimalClient {
private static final Logger LOGGER = LoggerFactory
.getLogger(RandomAnimalClient.class);
private final RestTemplate restTemplate;
private final DiscoveryClient discoveryClient;
RandomAnimalClient(RestTemplate restTemplate, DiscoveryClient discoveryClient) {
this.restTemplate = restTemplate;
this.discoveryClient = discoveryClient;
}
public ResponseEntity<Animal> random() {

        ServiceInstance instance = discoveryClient.getInstances("random-animal")
                .stream().findAny()
                .orElseThrow(() -> new IllegalStateException("Random-animal service unavailable"));
        UriComponentsBuilder uriComponentsBuilder = UriComponentsBuilder
                .fromHttpUrl(instance.getUri().toString() + "/random");
        return restTemplate.getForEntity(uriComponentsBuilder.toUriString(), Animal.class);
    }
}
DiscoveryClient получает список экземпляров микросервиса random-animal по его имени. Мы выбираем случайный экземпляр и обращаемся к нему.

@Loadbalanced RestTemplate делает примерно то же самое, но алгоритм выбора экземпляра лучше, так что рекомендуется использовать его.

С 2015 по умолчанию в Spring Cloud был включен балансировщик Ribbon от Netflix. Но сейчас есть новый Spring Cloud Load balancer, именно поэтому в настройках мы отключаем старый, как уже было показано выше:

spring.cloud.loadbalancer.ribbon.enabled=false
Итоги
Исходный код доступен на GitHub
https://github.com/myluckagain/sysout/tree/master/cloud1
. В следующей статье рассмотрим API Gateway — балансировку на стороне сервера.

Есть также о Circuit Breaker и Config-сервере.

