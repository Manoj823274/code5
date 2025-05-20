<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>weatherapp</artifactId>
    <version>1.0</version>
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- H2 Database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
        </dependency>

        <!-- Swagger -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!-- JSON -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
---------------------------------------------------------------------------
server:
  port: 8080

spring:
  datasource:
    url: jdbc:h2:mem:weatherdb
    driverClassName: org.h2.Driver
    username: sa
    password: 
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

weather:
  api-key: YOUR_OPENWEATHERMAP_API_KEY
  url: https://api.openweathermap.org/data/2.5/weather
------------------------------------------------------------------------------------------
@Entity
public class WeatherHistory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String city;
    private String temperature;
    private String description;
    private LocalDateTime dateTime;

    // Getters and Setters
}
-----------------------------------------------------------------
public interface WeatherHistoryRepository extends JpaRepository<WeatherHistory, Long> {
    List<WeatherHistory> findByCity(String city);
}
---------------------------------------------------------------
@Service
public class WeatherService {

    @Value("${weather.api-key}")
    private String apiKey;

    @Value("${weather.url}")
    private String apiUrl;

    private final RestTemplate restTemplate = new RestTemplate();
    private final WeatherHistoryRepository historyRepo;

    public WeatherService(WeatherHistoryRepository historyRepo) {
        this.historyRepo = historyRepo;
    }

    @Cacheable("weather")
    public Map<String, Object> getWeather(String city) {
        String url = String.format("%s?q=%s&appid=%s&units=metric", apiUrl, city, apiKey);
        ResponseEntity<Map> response = restTemplate.getForEntity(url, Map.class);

        Map<String, Object> body = response.getBody();
        saveHistory(city, body);
        return body;
    }

    public void saveHistory(String city, Map<String, Object> data) {
        WeatherHistory wh = new WeatherHistory();
        wh.setCity(city);
        wh.setTemperature(String.valueOf(((Map<String, Object>) data.get("main")).get("temp")));
        wh.setDescription(((Map<String, Object>) ((List<?>) data.get("weather")).get(0)).get("description").toString());
        wh.setDateTime(LocalDateTime.now());
        historyRepo.save(wh);
    }

    public List<WeatherHistory> getHistory(String city) {
        return historyRepo.findByCity(city);
    }
}
--------------------------------------------------------------------------
@RestController
@RequestMapping("/api/weather")
public class WeatherController {

    private final WeatherService service;

    public WeatherController(WeatherService service) {
        this.service = service;
    }

    @GetMapping("/{city}")
    public ResponseEntity<?> getWeather(@PathVariable String city) {
        return ResponseEntity.ok(service.getWeather(city));
    }

    @GetMapping("/history/{city}")
    public ResponseEntity<?> getHistory(@PathVariable String city) {
        return ResponseEntity.ok(service.getHistory(city));
    }
}
-----------------------------------------------------------------------------------
@SpringBootApplication
@EnableCaching
@EnableScheduling
public class WeatherApp {
    public static void main(String[] args) {
        SpringApplication.run(WeatherApp.class, args);
    }
}
--------------------------------------------------------------------------
@Component
public class WeatherCacheEvictor {

    @Scheduled(fixedRate = 3600000) // every hour
    @CacheEvict(value = "weather", allEntries = true)
    public void clearCache() {
        System.out.println("Weather cache cleared");
    }
}
