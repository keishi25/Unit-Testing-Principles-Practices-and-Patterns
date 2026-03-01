10.1.4 の状態ベースと移行ベースについて

下記で読み替えると理解しやすかった。
状態ベースは、state-based
移行ベースは、migration-based

現状の、スキーマ管理が移行ベースなので、特に状態ベースについては把握だけで十分

10.2.1 データ操作とトランザクション管理
本番コードの話
データ操作後、トランザクション管理する必要があるという話。
できていると思う。（実際のコードまだ見ていない）
観点：@transactionが適切に使われているか


10.2.2　データ操作とトランザクション管理
テストケースの話
テスト実行ときに、各フェーズでセッションを使いまわさない

理由
「ORマッパー（Hibernate/JPA）を使っているプロジェクトの結合テストで、セッションを使い回すと、テスト不備が発生する」
ITの自動化に即して、springbootTestの例記載。

❌ 問題が起きるケース（セッション使い回し）
@SpringBootTest
@Transactional // ← これが使い回しの原因になりやすい
class UserControllerIT extends Specification {

    @Autowired
    EntityManager entityManager

    @Autowired
    UserController sut

    def "メールアドレスを変更できる"() {
        given:
        def user = new User(email: "old@example.com", type: UserType.EMPLOYEE)
        entityManager.persist(user)
        entityManager.flush()

        when:
        sut.changeEmail(user.id, "new@gmail.com")

        then:
        def userFromDb = entityManager.find(User, user.id)
        // ↑ キャッシュから返るのでDBを確認していない
        userFromDb.email == "new@gmail.com" // 💥 偽陽性
    }
}

キャッシュ呼び出しになっているため、メールアドレスの変更が正しくテストできていない状態になっている。
Mybatis使っているので、上記問題は発生しないが、分けておくのがベター
