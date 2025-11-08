---
title: "Spring Bootのアノテーション概要 - 主要なアノテーションを体系的に理解する"
emoji: "☕"
type: "tech"
topics: ["SpringBoot", "Java", "アノテーション", "フレームワーク"]
published: true
---

## はじめに

Spring Bootを学び始めたとき、数多くのアノテーションに戸惑った経験はありませんか？  
`@SpringBootApplication`、`@RestController`、`@Autowired`...  
それぞれの役割はなんとなくわかるけれど、全体像がつかめない。

この記事では、Spring Bootでよく使われるアノテーションを**カテゴリ別に整理**して、  
体系的に理解できるようにまとめました。

---

## アノテーションとは

アノテーション（注釈）は、Javaのコードに**メタデータを付与する**ための仕組みです。  
Spring Bootでは、アノテーションを使って以下のような情報を表現します：

- このクラスは何をするものか（コンポーネント、コントローラーなど）
- どのように依存関係を解決するか（依存性注入）
- どのようにHTTPリクエストを処理するか（ルーティング）

---

## 1. アプリケーション設定系

### @SpringBootApplication

Spring Bootアプリケーションの**エントリーポイント**を表すアノテーション。

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

このアノテーションは、実は以下の3つのアノテーションを組み合わせたものです：
- `@Configuration` - 設定クラスであることを示す
- `@EnableAutoConfiguration` - 自動設定を有効化
- `@ComponentScan` - コンポーネントのスキャンを有効化

### @Configuration

**設定クラス**であることを示すアノテーション。  
`@Bean`メソッドを定義して、SpringコンテナにBeanを登録できます。

```java
@Configuration
public class AppConfig {
    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

### @EnableAutoConfiguration

Spring Bootの**自動設定機能**を有効化します。  
クラスパス上の依存関係を検出して、自動的に設定を行います。

---

## 2. コンポーネント登録系

### @Component

Springが管理する**汎用的なコンポーネント**であることを示します。  
このアノテーションが付いたクラスは、自動的にSpringコンテナに登録されます。

```java
@Component
public class UserService {
    // ビジネスロジック
}
```

### @Service

**サービス層**のコンポーネントであることを示します。  
実体は`@Component`と同じですが、**意図を明確にする**ために使用します。

```java
@Service
public class UserService {
    public User findById(Long id) {
        // ユーザー検索ロジック
    }
}
```

### @Repository

**データアクセス層**のコンポーネントであることを示します。  
データベースアクセスに関する例外を、Springの統一例外に変換してくれます。

```java
@Repository
public class UserRepository {
    public User findById(Long id) {
        // データベースアクセス
    }
}
```

### @Controller

**MVCのコントローラー**であることを示します。  
主にビューを返す場合に使用します。

```java
@Controller
public class UserController {
    @GetMapping("/users")
    public String listUsers(Model model) {
        // ユーザー一覧を取得してビューに渡す
        return "users/list";
    }
}
```

---

## 3. Web開発系（REST API）

### @RestController

**REST APIのコントローラー**であることを示します。  
`@Controller`と`@ResponseBody`を組み合わせたアノテーションで、  
戻り値がJSONやXMLなどのレスポンスボディとして返されます。

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

### @RequestMapping

**URLパターンとHTTPメソッド**をマッピングします。  
クラスレベルとメソッドレベルの両方で使用できます。

```java
@RequestMapping("/api/users")
public class UserController {
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public User getUser(@PathVariable Long id) {
        // ...
    }
}
```

### @GetMapping / @PostMapping / @PutMapping / @DeleteMapping

`@RequestMapping`の**ショートカットアノテーション**です。  
HTTPメソッドごとに専用のアノテーションが用意されています。

```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")      // GET /users/{id}
    public User getUser(@PathVariable Long id) { }
    
    @PostMapping("/users")           // POST /users
    public User createUser(@RequestBody User user) { }
    
    @PutMapping("/users/{id}")       // PUT /users/{id}
    public User updateUser(@PathVariable Long id, @RequestBody User user) { }
    
    @DeleteMapping("/users/{id}")    // DELETE /users/{id}
    public void deleteUser(@PathVariable Long id) { }
}
```

### @PathVariable

URLパスから**変数を取得**します。

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    // /users/123 にアクセスした場合、id = 123
}
```

### @RequestParam

**クエリパラメータ**から値を取得します。

```java
@GetMapping("/users")
public List<User> searchUsers(@RequestParam String name) {
    // /users?name=田中 にアクセスした場合、name = "田中"
}
```

### @RequestBody

**リクエストボディ**（JSONなど）をオブジェクトに変換します。

```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // POSTリクエストのボディをUserオブジェクトに変換
}
```

### @ResponseBody

戻り値を**レスポンスボディ**として返します。  
`@RestController`を使う場合は自動的に適用されるため、通常は不要です。

---

## 4. 依存性注入系

### @Autowired

Springコンテナから**自動的に依存関係を注入**します。  
コンストラクタ、フィールド、セッターに使用できます。

```java
@Service
public class UserService {
    private final UserRepository repository;
    
    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

**注意**: コンストラクタが1つだけの場合は、`@Autowired`を省略できます。

```java
@Service
public class UserService {
    private final UserRepository repository;
    
    // @Autowiredは省略可能
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

### @Qualifier

同じ型のBeanが複数ある場合、**どのBeanを注入するか指定**します。

```java
@Service
public class UserService {
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource dataSource;
}
```

### @Primary

同じ型のBeanが複数ある場合、**優先的に注入されるBean**を指定します。

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    public DataSource secondaryDataSource() {
        return new BasicDataSource();
    }
}
```

### @Value

**プロパティファイル**（`application.properties`など）から値を注入します。

```java
@Component
public class AppConfig {
    @Value("${app.name}")
    private String appName;
    
    @Value("${server.port:8080}")  // デフォルト値も指定可能
    private int serverPort;
}
```

---

## 5. データアクセス系

### @Entity

JPAの**エンティティクラス**であることを示します。

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
}
```

### @Table

エンティティがマッピングされる**テーブル名**を指定します。

### @Id

**主キー**であることを示します。

### @GeneratedValue

主キーの**生成戦略**を指定します。

### @Column

**カラム名**や**制約**を指定します。

### @Transactional

**トランザクション管理**を行います。  
メソッドが正常に完了したらコミット、例外が発生したらロールバックします。

```java
@Service
public class UserService {
    @Transactional
    public User createUser(User user) {
        // このメソッド内の処理は1つのトランザクションで実行される
        return userRepository.save(user);
    }
}
```

---

## 6. 設定・プロファイル系

### @Profile

特定の**プロファイル**（環境）でのみ有効になるBeanを指定します。

```java
@Configuration
@Profile("production")
public class ProductionConfig {
    @Bean
    public DataSource dataSource() {
        // 本番環境用の設定
    }
}
```

### @ConditionalOnProperty

**プロパティの値**に応じて条件付きでBeanを登録します。

```java
@Configuration
@ConditionalOnProperty(name = "feature.cache.enabled", havingValue = "true")
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        // キャッシュが有効な場合のみ
    }
}
```

---

## 7. バリデーション系

### @Valid

**バリデーション**を実行します。  
通常、`@RequestBody`と組み合わせて使用します。

```java
@PostMapping("/users")
public User createUser(@Valid @RequestBody User user) {
    // Userオブジェクトのバリデーションが実行される
}
```

### @NotNull / @NotBlank / @Size / @Email

**バリデーションルール**を指定します。

```java
public class User {
    @NotNull
    @NotBlank
    private String name;
    
    @Email
    private String email;
    
    @Size(min = 8, max = 20)
    private String password;
}
```

---

## 8. スコープ系

### @Scope

Beanの**スコープ**（生存期間）を指定します。

```java
@Component
@Scope("prototype")  // 毎回新しいインスタンスを作成
public class PrototypeBean {
    // ...
}
```

主なスコープ：
- `singleton`（デフォルト）- アプリケーション全体で1つのインスタンス
- `prototype` - 毎回新しいインスタンスを作成
- `request` - HTTPリクエストごとに1つのインスタンス
- `session` - HTTPセッションごとに1つのインスタンス

---

## アノテーションの使い分けのコツ

### 1. レイヤーごとに適切なアノテーションを使う

```
Controller層 → @RestController または @Controller
Service層    → @Service
Repository層 → @Repository
設定クラス    → @Configuration
```

### 2. 意図を明確にする

`@Component`でも動作しますが、`@Service`や`@Repository`を使うことで、  
コードを読む人に**意図が伝わりやすく**なります。

### 3. 依存性注入はコンストラクタ注入を推奨

フィールド注入よりも、**コンストラクタ注入**の方が：
- テストしやすい
- 不変性を保てる
- 必須の依存関係を明確にできる

```java
// 推奨
@Service
public class UserService {
    private final UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}

// 非推奨（フィールド注入）
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
}
```

---

## まとめ

Spring Bootのアノテーションは、大きく以下のカテゴリに分けられます：

1. **アプリケーション設定系** - `@SpringBootApplication`、`@Configuration`
2. **コンポーネント登録系** - `@Component`、`@Service`、`@Repository`、`@Controller`
3. **Web開発系** - `@RestController`、`@GetMapping`、`@PostMapping`など
4. **依存性注入系** - `@Autowired`、`@Qualifier`、`@Primary`
5. **データアクセス系** - `@Entity`、`@Transactional`
6. **設定・プロファイル系** - `@Profile`、`@ConditionalOnProperty`
7. **バリデーション系** - `@Valid`、`@NotNull`など
8. **スコープ系** - `@Scope`

最初は数が多くて大変ですが、**カテゴリごとに整理**して覚えると、  
理解が深まります。

実際にコードを書きながら、一つずつ覚えていきましょう！ 🚀

---

## 参考資料

- [Spring Boot公式ドキュメント](https://spring.io/projects/spring-boot)
- [Spring Framework公式ドキュメント](https://spring.io/projects/spring-framework)

