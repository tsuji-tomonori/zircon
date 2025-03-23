# 概要 (Overview)
ZirconはAIアシスト機能付きで無限に子タスクを生成できる高度なToDoアプリケーションです。本設計では、Zirconに求められる**柔軟かつ拡張可能な権限管理アーキテクチャ**を提案します。Zirconではユーザーがプロジェクトごとに**「メンバー」**・**「プロジェクト管理者」**・**「システム管理者」**のロールを持ち、それぞれのロールに応じて操作権限が異なります。さらに、プロジェクト内でユーザーやグループ単位に、タスクの**作成**・**編集**・**閲覧**・**ステータス変更**など細かなアクションごとの権限を制御できるようにします。本提案では、AWSが提供するポリシー言語**Cedar**を活用して、こうした**ロールベース + ポリシーベース**のきめ細かな認可(アクセス制御)を実現する方法を示します。バックエンドはTypeScript + Honoフレームワーク、データストアはPostgreSQL、インフラはAWS Lambda中心のサーバレス構成を前提にしています。以下、要件の整理からエンティティモデル設計、Cedarポリシーの記述例、システム全体のアーキテクチャ、およびCedar導入方法までを順に説明します。

# 要件整理 (Requirements)
まず、Zirconの権限管理に関する要件を整理します。

- **プロジェクトごとのロール**: ユーザーはプロジェクト単位で役割が異なります。同じユーザーでも、あるプロジェクトでは「メンバー」権限、別のプロジェクトでは「プロジェクト管理者」権限を持つ、といったシナリオがあり得ます。全システムに対する「システム管理者」ロールも存在します。
- **基本ロールの定義**:
  - *メンバー*: プロジェクト内で基本的なタスク操作（例: 自分のタスクの作成・編集・閲覧など）が可能。
  - *プロジェクト管理者*: 当該プロジェクト内の全タスクおよびプロジェクト設定を管理可能（例: 全メンバーのタスク閲覧・編集、タスクのステータス変更、メンバー招待など）。
  - *システム管理者*: システム全体の管理者で、すべてのプロジェクトやタスクに対する全権限（プロジェクトの新規作成や削除、全プロジェクトのタスク閲覧など）を持ちます。
- **詳細な操作権限の制御**: 単にロールだけでなく、各ユーザーまたはグループに対して**タスクの作成**・**編集**・**閲覧**・**完了状態への変更**など個別の操作権限を設定できるようにします。例えば、「プロジェクトAでは一般メンバーでもタスク作成を許可するが、プロジェクトBではメンバーは作成不可」といった柔軟な設定を可能にします。
- **ユーザーグループによる権限付与**: プロジェクト内に独自のユーザーグループを定義し、そのグループごとに権限を付与・制限できるようにします。これにより、「開発チーム」「デザインチーム」など任意のグループ単位でタスク操作権限を調整できます。
- **拡張性**: 新しいプロジェクトの追加やロールの追加、あるいは新しいアクション種別（例えば将来的に「コメント権限」などが増える）にも対応できるアーキテクチャとします。権限ルールの変更追加がコードの改変なしに可能であることが望ましいです。
- **一貫性と保守性**: 認可ロジックはアプリケーションロジックから分離し、一元管理します。複数のサービスやコンポーネント間で権限チェックの結果が統一されるよう、中央集権的なポリシー管理を行います。また、管理者が権限設定を監査・確認しやすいようにします。

以上の要件を満たすために、**ポリシーベースのアクセス制御**を導入し、柔軟な条件付きのルールで許可/拒否を定義できるようにします。

# アクセス制御方式の選定 (Choosing a Permission Management Approach)
Zirconのように権限要件が複雑化する場合、従来の静的なロールベースアクセス制御 (Role-Based Access Control: RBAC) だけでは細かな要件への対応が難しくなります。RBACではロールごとに固定的な権限集合を割り当てますが、本ケースでは**同じロール名でもプロジェクトによって権限が変わる**点や、**ロール横断的な例外**（特定ユーザーやグループにのみ追加許可を与える等）が求められます。こうした場合、**属性ベースアクセス制御 (Attribute-Based Access Control: ABAC)** や**ポリシーベースアクセス制御**の導入が有効です。

**Cedarポリシー言語**は、AWSが開発・提供するオープンソースの認可ポリシー言語で、RBACとABACの双方のモデルをサポートしつつ、人間に読み書きしやすい文法で権限ルールを記述できます ([Introducing Cedar, an open-source language for access control](https://aws.amazon.com/about-aws/whats-new/2023/05/cedar-open-source-language-access-control/#:~:text=Today%2C%20AWS%20open,the%20model%20matches%20the%20Rust)) ([Introducing Cedar, an open-source language for access control](https://aws.amazon.com/about-aws/whats-new/2023/05/cedar-open-source-language-access-control/#:~:text=Amazon%20Verified%20Permissions%20uses%20Cedar,are%20disconnected%20from%20the%20network))。Cedarを使うことで以下のメリットが得られます。

- **ポリシーによる柔軟な表現力**: 「誰 (principal) が、何のアクション (action) を、どのリソース (resource) に対して行えるか」をポリシーとして定義できます。条件節を用いれば属性に基づいた細かな条件（例えば「プロジェクトXの管理者ロールを持つ場合のみ」等）を組み込むことも可能です。
- **RBAC+ABACの組み合わせ**: Cedarポリシーでは、ロール（グループ）単位の許可を定義しつつ、必要に応じて属性ベースの条件を追加できます ([Amazon Verified Permissions と Cedar ポリシー言語の用語と概念](https://docs.aws.amazon.com/ja_jp/verifiedpermissions/latest/userguide/terminology.html#:~:text=Cedar%20%E3%81%AE%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E8%A8%80%E8%AA%9E%E3%81%A7%E3%81%AF%E3%80%81%E5%B1%9E%E6%80%A7%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AE%E6%9D%A1%E4%BB%B6%E3%82%92%E6%8C%81%E3%81%A4%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E3%81%AB%E5%AF%BE%E3%81%97%E3%81%A6%E6%A8%A9%E9%99%90%E3%82%92%E5%AE%9A%E7%BE%A9%E3%81%A7%E3%81%8D%E3%82%8B%E3%81%9F%E3%82%81%E3%80%81RBAC%20%E3%81%A8%20ABAC%20%E3%82%92,1%20%E3%81%A4%E3%81%AE%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%AB%E3%81%BE%E3%81%A8%E3%82%81%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82%20%E8%AA%8D%E5%8F%AF))。これにより「基本はロールで管理し、例外的な細則は属性条件で補う」といった一体的な権限管理が可能です。
- **アプリケーションロジックからの分離**: 認可ロジックをコードから切り離し、ポリシーファイルや外部ストアで集中管理できます。これにより、権限変更時にアプリケーションの再デプロイなしでポリシーの追加・更新ができます（たとえば新しいプロジェクトやグループに対するルールを追加する場合など）。
- **検証とテストの容易さ**: Cedarにはポリシーの検証ツールやテストフレームワークが用意されており、誤ったルールによる権限漏れを事前にチェックできます。また、ポリシーはテキスト（またはJSON）で表現されるためコードレビューもしやすく、監査ログに残すことも容易です。
- **AWS環境との親和性**: CedarはAWSのマネージドサービス**Amazon Verified Permissions**においてネイティブに採用されています ([Introducing Cedar, an open-source language for access control](https://aws.amazon.com/about-aws/whats-new/2023/05/cedar-open-source-language-access-control/#:~:text=Amazon%20Verified%20Permissions%20uses%20Cedar,are%20disconnected%20from%20the%20network))。AWS環境下であれば、Verified Permissionsサービスを利用してCedarポリシーの保存・評価を行うことで、自前で認可エンジンを構築することなく高スループットかつ低レイテンシな認可チェックを実現できます。また、オープンソース版のCedarエンジンも公開されており（Rust実装およびWASMビルド）、AWS外の環境やカスタム要件にも対応可能です。

以上の理由から、本システムでは**Cedarによるポリシーベースのアクセス制御**を採用します。基本的にはRBACで定義したロール（メンバー、プロジェクト管理者、システム管理者）をCedar上で「グループ」として表現し、それらに対する許可ポリシーを定義します。さらに、プロジェクト内グループや個人単位の細かな権限も追加のポリシーで定義し、RBACでは表現しづらい例外に対応します。認可クエリの評価にはAWSのVerified Permissionsサービスを利用することを想定しつつ、必要に応じてCedarエンジンの直接組み込みも検討します。

# エンティティモデル設計 (Entity Model Design)
次に、権限管理に関連する**エンティティ（データモデル）**の設計を示します。ここでは、アプリケーションのDBモデルとCedarポリシーにおけるエンティティ表現の両面から説明します。

**1. ユーザー (User)**:
アプリ利用者を表すエンティティです。PostgreSQL上では`users`テーブルを持ち、ユーザーID、名前やメールアドレス等を管理します。認可の観点では各ユーザーが様々なプロジェクトに属し、ロールやグループのメンバーシップを持ちます。Cedarではユーザーをプリンシパルの一種として扱い、エンティティタイプ`User`として定義します（例えばCedar上で`User::"alice"`などと識別）。ユーザーエンティティには属性としてプロフィール情報を持たせることも可能ですが、権限判定では主に**所属するロール/グループ**を用います。

**2. プロジェクト (Project)**:
ToDoを管理する単位となるプロジェクトです。`projects`テーブルで管理し、各プロジェクトIDや名称、説明などを持ちます。各プロジェクトには複数のユーザーが参加し、それぞれのユーザーにロール（メンバー/管理者）が割り当てられます。Cedar上ではリソースのコンテナとして`Project`タイプのエンティティを定義します。プロジェクトエンティティは親子関係の**ルート**として機能し、この下にタスクが紐付きます。また、後述のようにCedar上で「プロジェクトごとのロールグループ」を表現する際にもProjectエンティティを参照します。

**3. タスク (Task)**:
ユーザーが管理・操作するToDo項目です。タスクは**無限に子タスクを持てる**という仕様上、親子構造（階層構造）を取ります。DB上では`tasks`テーブルに、各タスクのID、所属プロジェクトID、（親タスクID）、タスク名、内容、ステータス、担当者などの情報を格納します。トップレベルのタスクは親タスクIDがNULLでプロジェクトIDと結びつき、子タスクは親タスクIDを介してどのプロジェクトに属するか間接的に決まります（子孫のタスクはすべてルートのプロジェクトに属する）。Cedarでは、タスクをリソースエンティティ`Task`として表現し、**親子関係**をエンティティの関係性として持たせます。例えば、Cedarエンティティ定義上`Task`は`Project`または別の`Task`を親（parent）に持つことができる関係とします。これによりCedarポリシー内で「タスクが属するプロジェクト」を推論でき、プロジェクト単位の許可ルールをタスクにも適用できます。

**4. ロール (Role)**:
ユーザーの役割を表す概念上のエンティティです。本システムの基本ロールは「メンバー」「プロジェクト管理者」「システム管理者」の3種ですが、プロジェクトごとに割り当てる必要があるため、**プロジェクト固有のロールグループ**として扱います。すなわち、**「Project A の管理者」**と**「Project B の管理者」**は別グループとして管理します。DB上ではユーザーとプロジェクトのリレーションとして`project_memberships`テーブルなどを用意し、各ユーザーが各プロジェクトで持つロールを記録します（例: user_id=42がproject_id=100でrole="admin"）。Cedarではロールを**グループエンティティ**としてモデル化します。具体的には、エンティティタイプ`Role`を定義し、そのIDとして「プロジェクトIDとロール名の組み合わせ」を表現します。例えばプロジェクトIDが`proj123`のプロジェクトに対する管理者ロールは、Cedar上で`Role::"proj123_Admin"`のようなIDとします。同様にプロジェクト`proj123`のメンバーロールは`Role::"proj123_Member"`となります。各ユーザーは自分の属するロールグループを**親エンティティ**として持ちます（Cedarではエンティティに複数の親を設定可能です ([Implementing Role-Based Access Control (RBAC) with AWS’ Cedar](https://www.permit.io/blog/cedar-rbac#:~:text=In%20this%20example%2C%20we%20defined,in%20the%20following%20manner)) ([Implementing Role-Based Access Control (RBAC) with AWS’ Cedar](https://www.permit.io/blog/cedar-rbac#:~:text=The%20JSON%20format%20of%20an%C2%A0Entity,is))）。たとえばユーザーAliceがプロジェクト`proj123`では管理者、`proj456`ではメンバーであり、さらにシステム管理者でもある場合、Aliceのユーザーエンティティは以下のような親参照を持ちます。

```json
{
  "entityId": { "type": "User", "id": "alice" },
  "parents": [
    { "type": "Role", "id": "proj123_Admin" },
    { "type": "Role", "id": "proj456_Member" },
    { "type": "Role", "id": "SystemAdmin" }
  ],
  "attributes": { ... }
}
```

上記のように、`Role::"SystemAdmin"`はプロジェクトに紐付かない**グローバルロール**として扱います。一方、`projXXX_Admin`や`projXXX_Member`のようなロールIDは各プロジェクト固有のグループです。ポリシー側ではこれらロールグループに対する許可規則を定義し、ユーザーは所属グループに応じた権限が与えられます。なお、ロールのメンバーシップ（誰がどのRoleグループに属しているか）はポリシーストアに静的に保存するか、認可リクエスト時に動的に提供する必要があります（後述）。

**5. ユーザー定義グループ (Group)**:
プロジェクト内で任意に定義されるユーザーグループです。これは「ロール」とは別に、より自由な粒度でユーザー集合を表現するためのものです。例えばプロジェクト管理者が「設計チーム」「QA担当」などのグループを作成し、それぞれに特定のタスク操作権限を与えることができます。DB上では`groups`（または`project_groups`）テーブルにグループID、所属プロジェクトID、グループ名等を保持し、`group_members`テーブルでユーザーIDとグループIDの関係を管理します。Cedar上では、これらをロールと同様に**グループエンティティ**として扱います。区別のためエンティティタイプ名を`Group`とし、IDにはプロジェクトIDとグループ名を含めてユニークにします（例: プロジェクト`proj123`のグループ「Designers」は`Group::"proj123_Designers"`と表現）。ユーザーエンティティは所属するグループも親として持ちます（ロールとグループでタイプが異なっても、親に混在させて問題ありません）。グループに付与するカスタム権限は、Cedarポリシーとして個別に記述・管理します。

**6. アクション (Action)**:
アプリケーション内でユーザーが起こし得る操作の種類です。認可判定では、このアクションの種類に基づいて許可/不許可を判断します。本システムでは主要なアクションとして「プロジェクトの管理系操作」と「タスクの操作」に分類できます。具体的には:
  - プロジェクト管理系: プロジェクト自体の作成・削除、メンバー招待やロール変更、プロジェクト設定変更等。
  - タスク操作系: タスクの作成、編集（内容変更）、閲覧（読み取り）、ステータス変更（例: 完了・アーカイブ等）、削除など。
これらアクションはCedar上で`Action`という特別なエンティティタイプとして扱われ、ポリシー内で`action == Action::"ActionName"`の形で参照されます。スキーマ定義時にアクションの一覧（またはカテゴリー）を定義します。例えば`Action::"CreateTask"`, `Action::"EditTask"`, `Action::"ViewTask"`, `Action::"ChangeStatus"` 等を列挙します。共通するアクションは**アクショングループ**としてまとめることも可能です ([Using role-based access control | Cedar Policy Language Reference Guide](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles.html#:~:text=We%20can%20simplify%20this%20policy,TimeSheetApprove)) ([Using role-based access control | Cedar Policy Language Reference Guide](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles.html#:~:text=action%20in%20Action%3A%3A,timesheets%22))。例えば`Action::"TaskActions"`というグループにタスクに対する全操作を含めておけば、ポリシー記述を簡潔にすることができます。

**7. リソースグループ (Resource Group)**:
ポリシー記述を簡潔にするため、リソースの集合に名前を付けることもあります ([Using role-based access control | Cedar Policy Language Reference Guide](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles.html#:~:text=))。例えば全プロジェクトに含まれる全タスクを表す仮想的なグループ`TaskGrp::"all-tasks"`を定義し、すべてのタスクをこのグループに属するものと見做すことができます。Cedarではエンティティに複数の親を持たせられるため、各タスクエンティティの親の一つに`TaskGrp::"all-tasks"`を設定しておけば`resource in TaskGrp::"all-tasks"`という条件で「任意のタスク」を指すことができます。同様に全プロジェクトの集合`ProjectGrp::"all-projects"`を定義し、各Projectをその子にすれば`resource in ProjectGrp::"all-projects"`で全プロジェクトをカバーできます。本システムでは、システム管理者の権限付与など**全リソース対象**のポリシー記述にこのリソースグループを利用します（下記ポリシー例参照）。

以上が基本的なエンティティモデルです。これらを整理すると、Cedarのエンティティタイプとしては**User, Project, Task, Role, Group, (TaskGrp, ProjectGrp)**を定義することになります。また、それぞれの関係性をスキーマで定義します。例えば、スキーマ上で`entity Task { parent: Project, parent: Task }`のように記述し、タスクの親はプロジェクトか別のタスクであることを示します。同様に`entity User { parent: Role, parent: Group }`と定義し、ユーザーの親としてRoleまたはGroupが許容されるようにします。これにより、誤ったエンティティ関連（例えばユーザーの親にプロジェクトを直接付けるなど）を防止し、ポリシー評価時に型安全なデータを渡すことができます。

# ロールと基本権限の設計 (Roles and Base Permissions)
このセクションでは、基本ロール（メンバー、プロジェクト管理者、システム管理者）ごとのアクセス権と、それをCedarポリシーでどのように表現するかを示します。

**1. メンバー (Member)**:
デフォルトでは、プロジェクトのメンバーは**自身に関わるタスクの閲覧・編集・作成**など基本的な操作が可能です。ただしプロジェクト内の他ユーザーのタスク管理やプロジェクト全体の設定変更はできません。具体例として:
- タスク閲覧: メンバーはそのプロジェクト内のタスクを閲覧可能とします（ただし秘密のタスクがある場合など追加条件は別途検討）。
- タスク作成: 一般メンバーも自分の担当タスクを新規作成できる、とします（プロジェクト方針によっては管理者のみ作成にしたいケースもありますが、それは後述のグループ権限で制御可能とします）。
- タスク編集: 自分が作成したタスクや担当のタスクについて内容やステータスを更新可能とします。
- ステータス変更: タスクの完了/未完了、優先度変更などのステータス更新は、自分のタスクについて許可。
- プロジェクト管理操作: 招待・設定変更などは**不可**。

このようなメンバー権限をCedarで記述するには、**プロジェクト単位のメンバーグループ**に対する許可ポリシーとして表現します。例えば、プロジェクト`proj123`のメンバーがタスク閲覧・作成・編集をできるようにするポリシーは以下のようになります（Actionグループ`MemberActions`は仮に閲覧・作成・編集アクションをまとめたものとします）。

```cedar
// プロジェクトproj123のメンバーはそのプロジェクト内のタスク閲覧・作成・編集を許可
permit(
  principal in Role::"proj123_Member",
  action in Action::"MemberActions",
  resource in Project::"proj123"
);
```

上記では`principal in Role::"proj123_Member"`で「プロジェクト123のメンバーグループに属する主体」を表し、`resource in Project::"proj123"`で「リソースがプロジェクト123に属するタスクやリソース」であることを示しています。結果として、「proj123のメンバーであるユーザー」に対し「proj123内のタスク」に関する基本操作を許可する、というポリシーになります。

メンバーのポリシーは全プロジェクトで共通の構造ですが、プロジェクトIDごとに別個のポリシー文が必要になります。ただし、数多くのプロジェクトが存在する場合にポリシーが膨大になる点が懸念されます。その対策として、**ポリシーテンプレート**や**属性条件**を用いて一般化する方法も考えられます（後述）。ここではシンプルさを優先してプロジェクトごとにポリシーを用意する想定とします。

**2. プロジェクト管理者 (Project Admin)**:
プロジェクト管理者は自分の管轄するプロジェクト内で**ほぼすべての操作**が可能です。具体的には:
- 当該プロジェクト内の全タスクの閲覧・編集・作成・削除・ステータス変更が可能（自分のタスクかどうかに関わらず許可）。
- プロジェクト内のユーザー管理が可能（メンバー招待、ロール変更。ただしシステム管理者ロールの付与は不可などの制限は考慮）。
- プロジェクトの設定変更が可能（プロジェクト名の変更等）。
- プロジェクト内でのグループ作成や権限設定の管理が可能。

Cedarポリシーでは、プロジェクト管理者ロール（各プロジェクトごとの`Role::"projX_Admin"`グループ）に対して包括的な許可を与えます。タスク関連操作についてはメンバーより広範なアクションが許可され、プロジェクト管理操作についても許可します。例えば、プロジェクト`proj123`の管理者権限ポリシー例を示します。

```cedar
// プロジェクトproj123の管理者はそのプロジェクト内の全操作を許可
permit(
  principal in Role::"proj123_Admin",
  action in Action::"AllProjectActions",
  resource in Project::"proj123"
);
```

ここで`AllProjectActions`は当該プロジェクト内で管理者が実行し得る全アクション（タスクのCRUD、ステータス変更、プロジェクト設定変更等）をまとめたアクショングループだとします。`resource in Project::"proj123"`により、プロジェクト123自身およびその配下の全リソース（タスクなど）が対象となります。したがってこの1行で「プロジェクト123の管理者はプロジェクト123内のすべてのリソースに対する全操作を許可」という強力な規則を定義できます。

補足として、プロジェクト管理者には本来メンバーの権限も含まれるため、理想的には**ロールの継承**関係を表現すると管理が楽になります。Cedarではエンティティの親子関係を活用してロールの階層を表現できます ([Implementing Role-Based Access Control (RBAC) with AWS’ Cedar](https://www.permit.io/blog/cedar-rbac#:~:text=Image%3A%20Group%2067736))。例えば`Role::"proj123_Admin"`エンティティの親に`Role::"proj123_Member"`を設定しておけば、管理者はメンバーの権限も**自動的に継承**します。このようにしておくと、メンバー用のポリシーを一度書くだけで管理者にも適用されます（上記ポリシー例では管理者用に包括的な許可を書いているため直接は必要ないですが、継承関係を設定しておくことはポリシー管理上好ましいでしょう）。

**3. システム管理者 (System Admin)**:
システム管理者はZircon全体のスーパー管理者であり、**全プロジェクトに対する全操作**が許可されます。また、通常のプロジェクト範囲外の操作（例: 新規プロジェクトの作成/削除、他ユーザーへのシステム管理者権限付与、全システム設定の変更など）も行えます。Cedar上では、システム管理者はグローバルなロールグループ`Role::"SystemAdmin"`で表現し、これに対するポリシーを定義します。全プロジェクト・全タスクに対する許可を表現するために、前述のリソースグループ`ProjectGrp::"all-projects"`を活用します。例えば以下のようなポリシーになります。

```cedar
// システム管理者は全プロジェクト内の全リソースに対する全アクションを許可
permit(
  principal in Role::"SystemAdmin",
  action in Action::"AllActions",
  resource in ProjectGrp::"all-projects"
);
```

ここでは`AllActions`がアクションタイプ全てを網羅するグループ（またはワイルドカード的なもの）であるとします。`resource in ProjectGrp::"all-projects"`によって全てのプロジェクトおよびその配下のリソースが対象となります。従ってこのポリシーは**システム管理者なら何でもできる**ことを意味します。加えて、システム管理者だけが可能な特殊な操作（例えば「プロジェクト作成」という、まだ存在しないリソースに対する操作）は別途扱う必要がありますが、その場合はリソースを`ProjectGrp::"all-projects"`自体に対するアクションとして表現できます。例えば「プロジェクト作成」アクションはリソース`ProjectGrp::"all-projects"`（全プロジェクトのコンテナ）に対する`CreateProject`操作と見なすことで、上記ポリシーの適用範囲に含めることが可能です。もしくはシステム管理者用に個別に以下のようなポリシーを追加しても良いでしょう。

```cedar
// システム管理者はプロジェクトの作成・削除を許可
permit(
  principal in Role::"SystemAdmin",
  action in [Action::"CreateProject", Action::"DeleteProject"],
  resource == ProjectGrp::"all-projects"
);
```

以上が基本ロールに対するポリシーの例となります。なお、**ポリシーの効果**はデフォルトで許可denyベースとなっており、何のポリシーにもマッチしない要求は自動的に拒否されます ([Implementing Role-Based Access Control (RBAC) with AWS’ Cedar](https://www.permit.io/blog/cedar-rbac#:~:text=%2A%20Effect%C2%A0,by%20overriding%20a%20permit%20policy))。必要に応じて特定の場合に明示的拒否を与える`forbid`ポリシーも記述できます（permitより優先される）。例えば、「システム管理者でもプロジェクト削除は二人以上の承認が必要」といったルールを作る場合、`forbid`ポリシーで単独では削除不可とし、別途承認フローでpermitさせるなどの実装も可能ですが、本設計の範囲ではそこまで踏み込みません。

# グループと詳細権限の拡張 (Fine-Grained Permissions via Groups)
プロジェクト管理者は、基本ロールによる権限制御に加えて、**独自のユーザーグループ**を作成し、きめ細かな権限を設定できます。これにより各プロジェクト毎に柔軟なポリシー拡張が可能となります。ここではグループを用いた詳細権限の付与例と、その背後のモデルを解説します。

**ユーザーグループの作成とメンバー割当**: プロジェクト管理者はプロジェクト内に任意の名称でグループを作成できます。例えば「閲覧専用ユーザー」「外部協力者」「テスター」等、用途に応じたグループを作り、そのプロジェクト内のユーザーをメンバーとして追加します。これらグループはあくまでプロジェクト内の概念であり、別プロジェクトで同名のグループを作っても別物として扱われます。DB上は前述の`groups`テーブル（プロジェクトIDとグループ名でユニーク）と`group_members`テーブルで管理されます。Cedarでは`Group::"projX_GroupName"`というエンティティIDで表現され、各ユーザーは所属するGroupエンティティを親として持ちます。

**グループへの権限付与**: プロジェクト管理者はグループごとにタスク操作権限のON/OFFを細かく設定できます。例えば:
- 「閲覧専用ユーザー」グループにはタスクの閲覧だけ許可し、作成・編集・ステータス変更は不可にする。
- 「外部協力者」グループにはタスクの閲覧とコメント追加は許可するが、タスクの作成は不可にする（コメント機能があると仮定）。
- 「テスター」グループにはバグ報告用タスクの作成権限を与えるが、他のタスク編集は制限する。

これらは**追加のポリシー**としてCedarに登録します。グループ権限は基本ロールの許可を上書き・補完する役割です。例えば、基本ではメンバーにタスク作成が許可されていないプロジェクトで、一部ユーザーだけタスク作成を許可したい場合、「タスク作成権限を持つグループ」を作りそのユーザーを所属させることで対応します。その際Cedarポリシー上は当該グループに対し`CreateTask`アクションを許可するルールを追加します。具体例を示します。

例: プロジェクト`proj456`では通常メンバーはタスク作成不可とする一方、特定メンバーを「Contributor」グループに入れてタスク作成を許可する。

```cedar
// proj456のContributorグループはタスクの閲覧・作成を許可
permit(
  principal in Group::"proj456_Contributor",
  action in [Action::"ViewTask", Action::"CreateTask"],
  resource in Project::"proj456"
);
```

このポリシーによって、`proj456_Contributor`グループのユーザーはプロジェクト456内でタスク閲覧と作成が可能になります。他のメンバーは作成が許可されていないため（ベースポリシーで許可していなければデフォルト拒否）、グループに属するユーザーだけが追加の権限を持つことになります。

また別の例として、**例外的な個人に権限を与える**ことも可能です。Cedarポリシーでは`principal == User::"userID"`のように特定のユーザー個人を指定することもできます。そのため、グループを経由せず直接ユーザーに許可を書くことも技術的には可能です。ただし、ポリシーの保守性の観点から、個人単位の許可はできる限りグループ化して管理することを推奨します。グループを1人だけのために作成しても良いので、常に「グループまたはロールに対する許可」という形にするとポリシーの整理がしやすくなります。

**権限剥奪(拒否)の表現**: 基本的にPermitポリシーだけで必要十分な場合は、付与しないことで拒否を表現します。しかし高度な要件として「特定グループに属するユーザーはある操作だけ禁止（他の同じロールの人はできるがその人たちだけ不可）」というケースも考えられます。Cedarでは`forbid`ポリシーを使って特定条件下での拒否を明示できます。例えば、「外部協力者グループのユーザーはタスク削除は禁止」というルールは以下のように書けます。

```cedar
// proj456のExternalCollaboratorグループはタスク削除を禁止
forbid(
  principal in Group::"proj456_ExternalCollaborator",
  action == Action::"DeleteTask",
  resource in Project::"proj456"
);
```

`forbid`は対応する`permit`より優先されるため、仮に基本ロールや他のグループでDeleteTaskがpermitされていても、このグループに属するユーザーがDeleteTaskを実行しようとすると拒否されます。このように`permit`と`forbid`を組み合わせれば複雑な例外ケースにも対応可能ですが、運用上はできるだけシンプルなモデル（ロール + 必要なら追加グループによるpermit）で収まるようにするのが望ましいでしょう。

**ポリシー数と管理**: プロジェクトごと・グループごとにポリシーを追加していくと、システム全体で数百・数千のポリシーになる可能性があります。CedarおよびVerified Permissionsでは大量のポリシー管理にも耐えられるように設計されていますが、運用上は整理が必要です。例えば**ポリシーテンプレート**機能を活用すると、共通部分をテンプレート化しインスタンスごとにパラメータ（プロジェクトIDなど）を差し込んで登録することができます ([Roles with policy templates | Cedar Policy Language Reference Guide](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles-templates.html#:~:text=Roles%20with%20policy%20templates%20,for%20each%20of%20the%20roles)) ([Roles with policy templates | Cedar Policy Language Reference Guide](https://docs.cedarpolicy.com/bestpractices/bp-implementing-roles-templates.html#:~:text=You%20can%20manage%20role%20assignment,for%20each%20of%20the%20roles))。Verified PermissionsではポリシーテンプレートAPIが用意されており、「プロジェクトID」を変数にしたテンプレートを1つ書いて各プロジェクト用にインスタンス化するといった管理も検討できます。これにより同じ構造のポリシーを大量に記述する手間を省き、ルールの一貫性も保ちやすくなります。

# システムアーキテクチャとCedar導入方法 (System Architecture and Cedar Integration)
最後に、上記の権限モデルを実際のシステムに組み込み運用するためのアーキテクチャについて説明します。バックエンドはTypeScript + Hono、AWS Lambda上で稼働し、PostgreSQLに接続する構成を想定しています。認可サービスとしてAmazon Verified Permissionsを使用し、Cedarポリシーの管理・評価を委譲します（必要に応じてCedarエンジンを直接組み込むパターンにも触れます）。以下、ポリシーの管理方法、認可チェックの流れ、実装上の留意点について述べます。

## ポリシーストアの管理 (Policy Storage and Management)
Cedarポリシーの定義とエンティティ情報は、**ポリシーストア**として集中管理します。Amazon Verified Permissions (以下AVP) を利用する場合、まずAWS上にポリシーストアを作成し、そこにCedarの**スキーマ**と**ポリシー**を登録します。スキーマには前述したエンティティタイプ（User, Project, Task, Role, Group...）やアクション定義、エンティティ間の関係性ルールを記述します。ポリシーはCedar言語で書いたテキスト（またはJSON形式）をそのまま登録できます。通常、静的な基本ポリシー（例えば全プロジェクト共通のテンプレートポリシーやシステム管理者ポリシー）はアプリケーションデプロイ時にあらかじめストアにロードしておきます。一方、プロジェクト追加やグループ権限変更に伴う新規ポリシーや更新は、バックエンドからAVPのAPIを呼び出して動的に変更します。AVPではポリシーの追加・削除APIが提供されており、例えば「新しいプロジェクトが作成されたらそのプロジェクト向けのメンバー/管理者ポリシーを生成して登録」「管理者がGUIでグループの権限変更を行ったら対応するポリシーを更新」といったことが可能です ([What is Amazon Verified Permissions? - Amazon Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html#:~:text=Verified%20Permissions%20is%20a%20service,whether%20an%20action%20is%20permitted)) ([What is Amazon Verified Permissions? - Amazon Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html#:~:text=client%20application%20calls%20authorization%20APIs,whether%20an%20action%20is%20permitted))。これら操作はAWS SDK for JavaScript(TypeScript)をLambda上から呼び出すことで実現できます。コード例として、新規ポリシーを登録する場合は次のようなAWS SDK呼び出しになります（擬似コード）:

```typescript
import { VerifiedPermissionsClient, CreatePolicyCommand } from "@aws-sdk/client-verifiedpermissions";

const vpClient = new VerifiedPermissionsClient({ region: "ap-northeast-1" });
await vpClient.send(new CreatePolicyCommand({
  policyStoreId: "<ポリシーストアID>",
  policyType: "STATIC",  // 静的ポリシーとして登録
  policyDefinition: {
    "statements": `permit(
         principal in Role::"proj123_Member",
         action in Action::"MemberActions",
         resource in Project::"proj123"
     );`
  },
  // ↑policyDefinitionは本来JSON構造でも指定可。ここでは簡略化のため直接文字列を埋め込んでいます。
}));
```

上記はAWS SDK v3の例ですが、AVPのAPIエンドポイントに対しHTTPリクエストを送る形でも利用できます。登録されたポリシーは即座にポリシーストアに反映され、以降の認可判断に使用されます。

エンティティ（ユーザーやプロジェクト、ロールグループなど）の情報管理については2通りのアプローチが可能です。

- **(A) Verified Permissionsにエンティティも登録**: AVPは**エンティティストア**としての機能も持っており、各エンティティの属性や親関係を保存できます。例えばユーザーAliceに親Roleとしてproj123_Adminを設定したエンティティデータをAVPに保持させておけば、認可リクエストの際に逐一その情報を渡さなくても内部で参照して評価できます。この方式では、ユーザーのプロジェクト参加やロール変更があった場合に都度AVP側のエンティティデータを更新する必要があります。APIとしては`CreateEntity`, `UpdateEntity`等が用意されています。運用上データの二重管理になる点に注意が必要ですが、小〜中規模であればAVPのエンティティ管理を信頼してシンプルに構築できます。

- **(B) アプリケーション側でエンティティ情報を管理**: 既存のPostgreSQLに全情報がある場合、それを都度AVPに伝えて評価させる方法です。AVPの`IsAuthorized`（認可判定）API呼び出し時に、「今回のprincipal（ユーザー）やresource（タスク）に関するエンティティリスト」をリクエストに含められます ([Amazon Verified Permissions と Cedar ポリシー言語の違い](https://docs.aws.amazon.com/ja_jp/verifiedpermissions/latest/userguide/terminology-differences-avp-cedar.html#:~:text=Amazon%20Verified%20Permissions%20%E3%81%A8%20Cedar,%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E8%A8%80%E8%AA%9E%E3%81%AE%E9%81%95%E3%81%84%20Cedar%20%E3%81%AE%E8%AA%8D%E5%8F%AF%E6%96%B9%E6%B3%95%E3%81%A7%E3%81%AF%E3%80%81%E8%AA%8D%E5%8F%AF%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%82%92%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%A8%E7%85%A7%E3%82%89%E3%81%97%E5%90%88%E3%82%8F%E3%81%9B%E3%81%A6%E8%A9%95%E4%BE%A1%E3%81%99%E3%82%8B%E9%9A%9B%E3%81%AB%E8%80%83%E6%85%AE%E3%81%99%E3%81%B9%E3%81%8D%E3%82%A8%E3%83%B3%E3%83%86%E3%82%A3%E3%83%86%E3%82%A3%E3%81%AE%E3%83%AA%E3%82%B9%E3%83%88%E3%81%8C%E5%BF%85%E8%A6%81%E3%81%A7%E3%81%99%E3%80%82%20%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%8C%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E3%82%A2%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8))。この仕組みを使い、たとえばユーザーAliceがproj123の管理者でtask789を編集する場合、`User::"alice"`と`Role::"proj123_Admin"`の関係、`Task::"task789"`と`Project::"proj123"`の関係をエンティティリストとしてリクエストに渡します。AVPは受け取ったエンティティ情報を元にポリシーを評価し、許可/拒否を返します。こちらの方式では都度情報を提供するためリアルタイム性はありますが、認可リクエストが多少大きくなります。ただ、PostgreSQLとの整合性を取りやすく、アプリ側のトランザクション中の未確定データに基づく判定（例えばまだDBに書き込む前の新タスクに対する権限確認等）も可能になる利点があります。

本システムでは、シンプルさを優先し**(A)のエンティティ事前登録**＋**認可時はID指定のみ**の方式を採用します。つまりユーザーやプロジェクト、ロールメンバーシップなどはイベントドリブンでAVPに同期し、`IsAuthorized`を呼ぶ際にはprincipalやresourceのIDを渡すだけで済むようにします。例えばユーザーがプロジェクトに参加した際にそのユーザーエンティティにRole親を追加するAPIを呼び出す、といった実装になります。これにより認可チェック時のパフォーマンスが向上し（AVP側でキャッシュされたデータを即時評価できるため）、Lambda関数内の処理も簡潔になります。

ポリシーやエンティティ定義そのものはコード（インフラ構成管理）として管理しつつ、動的部分はアプリからAVP APIで操作する形となります。Cedarポリシーファイル一式もリポジトリで管理し、変更があればデプロイ時にAVPへ適用する運用を想定します（例えばPulumiやAWS CDKでポリシー登録をインフラコード化することもできます）。

## 認可リクエスト評価フロー (Authorization Check Flow)
アプリケーション上で実際に認可チェックを行う際の流れを示します。Honoフレームワーク上に**PEP (Policy Enforcement Point)**を実装し、AVP(Cedar)が**PDP (Policy Decision Point)**として機能する構成です ([Design models for Amazon Verified Permissions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/using-avp.html#:~:text=1,request%20data%2C%20including%20any%20JWTs)) ([Design models for Amazon Verified Permissions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/using-avp.html#:~:text=following%20diagram%20shows%20how%20you,tenant%20SaaS%20application))。

1. **ユーザー認証の完了**: クライアントからのAPIリクエストはまず認証を経てユーザーIDが確定します。Zirconでは例えばJWTによる認証を使っているとすると、Lambda上のHonoアプリでJWTを検証しユーザーを特定します（認証自体はCognito等で行い、ここでは結果として`userId`が得られている状態とします）。

2. **リクエストからアクション・リソースを抽出**: Honoのルーティングによりどのエンドポイントに対するどの操作かが分かります。例えば`POST /projects/:projectId/tasks`というエンドポイントに対するリクエストであれば、「プロジェクト:projectIdにタスク作成」の意図と判断できます。このようにエンドポイントとHTTPメソッドから**アクション種別**と**対象リソース**を決定します。設計上、各APIエンドポイントに対して想定されるCedar上の`Action`と`Resource`をあらかじめマッピングしておきます。Honoではミドルウェアを使ってリクエスト前処理ができるので、例えば以下のような関数を用意します。

   ```typescript
   // 認可ミドルウェアの例
   async function authorize(action: string, getResource: (c: Context) => {type: string, id: string}) {
     return async (c: Context, next: Function) => {
       const userId = c.get('userId');  // 認証済みユーザーIDをコンテキストから取得
       const resource = getResource(c); // コンテキストからリソースID等を抽出
       const allowed = await checkPermission(userId, action, resource);
       if (!allowed) {
         return c.text("Forbidden", 403);
       }
       await next();
     };
   }
   ```

   上記の`authorize`ミドルウェアは、与えられた`action`（例えば"CreateTask"）と、コンテキストからリソース情報を取り出す関数を受け取り、`checkPermission`関数（後述）で許可判定を行います。許可されなければ即座に403応答を返し、許可された場合のみ次のハンドラへ進みます。

3. **認可チェック (PDPへの問い合わせ)**: `checkPermission(userId, action, resource)`の中でAVPサービスに対し認可判定のAPI呼び出しを行います。AWS SDKでは`isAuthorized`または`IsAuthorized`コマンドが提供されています ([What is Amazon Verified Permissions? - Amazon Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html#:~:text=CloudFormation,whether%20an%20action%20is%20permitted))。このリクエストに、現在のユーザーをプリンシパル、アクションを今回の操作、リソースを対象リソースとして渡します。エンティティ情報は(先述の選択により)事前登録されている前提なので、ID指定のみ行います。擬似コードで示すと次のようになります。

   ```typescript
   import { IsAuthorizedCommand } from "@aws-sdk/client-verifiedpermissions";

   async function checkPermission(userId: string, action: string, resource: {type: string, id: string}): Promise<boolean> {
     const cmd = new IsAuthorizedCommand({
       policyStoreId: "<ポリシーストアID>",
       principal: { entityType: "User", entityId: userId },
       action:    { actionType: "Action", actionId: action },
       resource:  { entityType: resource.type, entityId: resource.id }
       // エンティティ情報は事前登録されている想定。未登録の場合はここに entities: [ {...}, {...} ] を含める。
     });
     const result = await vpClient.send(cmd);
     return result.decision === "Allow";
   }
   ```

   AVPは内部で該当するCedarポリシーを検索し、この(principal, action, resource)にマッチするルールがあるか評価します ([What is Amazon Verified Permissions? - Amazon Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html#:~:text=Verified%20Permissions%20provides%20authorization%20by,and%20how%20they%20were%20authenticated))。ロールやグループの所属関係もエンティティストアに基づいて考慮されます。例えば、`principal`が`User::"alice"`、`action`が`Action::"CreateTask"`、`resource`が`Task::"task789"`（そのTaskのparentはProject::"proj456"）というリクエストに対し、AVPはまず`alice`が属するRoleやGroupをエンティティストアから取得し、Aliceが`Role::"proj456_Admin"`を親にもつなら`principal in Role::"proj456_Admin"`のポリシーが適用できるか評価します。またTaskの親Projectがproj456であることから、`resource in Project::"proj456"`条件にも合致するか確認します。全て満たせばそのポリシーによって許可となります。もし複数のポリシーが該当する場合、許可ポリシーが1つでもあれば許可ですが、forbidポリシーがどれか一つでもマッチすれば強制拒否となります。

4. **結果に応じたレスポンス**: 認可結果がAllowならば、後続の実処理（例えばタスク作成処理など）を行い正常レスポンスを返します。Denyならば処理を中断し、HTTP 403 Forbiddenエラーを返します。これにより、権限のない操作はサーバレス関数内で早期に遮断されます。ログにはどのポリシーに基づいて拒否したか等を残すこともできます（AVPから「どのポリシーIDにマッチしたか」を取得するAPIもあります）。

このフローを各APIエンドポイントについて適用することで、一貫した認可チェックが実現できます。Honoのルーティング定義時に適宜`authorize`ミドルウェアを差し込んでおけば、アプリケーションロジック側は権限の存在を意識せずに記述できます（認可はアウトオブバンドで処理）。認可チェックにかかるレイテンシはAVPへのネットワーク呼び出し分だけですが、公式によればミリ秒オーダーで応答が返ります ([Introducing Cedar, an open-source language for access control](https://aws.amazon.com/about-aws/whats-new/2023/05/cedar-open-source-language-access-control/#:~:text=Amazon%20Verified%20Permissions%20uses%20Cedar,are%20disconnected%20from%20the%20network))。Lambda関数から同一リージョンのAVPを呼ぶ場合数～十数ミリ秒程度が見込まれ、許容範囲です。大量リクエスト時にもAVPはスケーラブルに裁いてくれるため、バックエンド側でキャッシュ等を行わずとも問題ありません。もしオフライン環境やレイテンシ極小化の要件がある場合、次の「Cedarエンジン直接組み込み」パターンも検討可能です。

## Cedarエンジン直接利用の代替 (Using Cedar SDK as an Alternative)
上記ではAWSのマネージドサービスを利用しましたが、Cedarはオープンソースエンジンとして自前でホストすることもできます。特にLambda環境では、外部サービスコールを削減するためにWASM版のCedarエンジンを関数内に組み込む方法があります。NPMパッケージ`@cedar-policy/cedar-wasm` ([@cedar-policy/cedar-wasm - npm](https://npmjs.com/package/@cedar-policy/cedar-wasm#:~:text=%40cedar))を使うと、Node.js/TypeScriptからCedarポリシーの評価を行うことができます。この場合、ポリシーとエンティティデータの保管・管理も自前で行う必要があります。例えばLambdaの初期化時にPostgreSQLから最新のポリシーセットを読み込んだり、あるいはポリシーファイルをコードにバンドルしておきCedarエンジンにロードする形です。評価自体はライブラリ内で完結するためネットワーク遅延はありませんが、Lambdaインスタンスがコールドスタートするたびにポリシー読み込みやWASM初期化の処理が発生する点に留意が必要です。

AWS Prescriptive Guidanceでも、Verified Permissionsを使わずCedar SDKを組み込む構成は**「カスタムPDPを自前で運用するモデル」**として触れられています ([Design models for Amazon Verified Permissions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/using-avp.html#:~:text=Using%20the%20Cedar%20SDK)) ([Design models for Amazon Verified Permissions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/using-avp.html#:~:text=store%20Cedar%20policies%20in%20a,because%20of%20inconsistent%20internet%20connectivity))。Verified Permissionsが利用できない特殊環境下（インターネット非接続など）では有効な代替ですが、その場合はポリシーストアの高可用性や一貫性を自前で担保する実装が必要です。本システムの前提ではAWSインフラを活用できるため、基本はVerified Permissionsを採用し、将来的な要件によってはCedarエンジン組み込みにスイッチできるよう抽象化しておく、というスタンスを取ります。具体的には、`checkPermission`関数の内部実装を切り替え可能にし、現在はAVP呼び出しだが将来WASMエンジン呼び出しに差し替えられるようにしておきます。例えば以下のように書いておくと良いでしょう。

```typescript
let cedarAuthorizer: CedarAuthorizer | null = null;

// Cedarエンジン初期化（WASM）関数: 必要時に呼び出す
async function initCedarEngine() {
  const cedar = await import("@cedar-policy/cedar-wasm");
  cedarAuthorizer = new cedar.Authorizer(policiesJsonArray, entitiesJsonArray, schemaJson);
}

// 汎用の認可関数: 内部でAVPまたはローカルエンジンを使用
async function checkPermission(userId: string, action: string, resource: {type:string, id:string}): Promise<boolean> {
  if (USE_AVP) {
    // 上述のAVP呼び出し
    ...
  } else {
    if (!cedarAuthorizer) {
      await initCedarEngine();
    }
    const request = {
      principal: { "entityType": "User", "entityId": userId },
      action:    { "actionType": "Action", "actionId": action },
      resource:  { "entityType": resource.type, "entityId": resource.id }
    };
    const decision = cedarAuthorizer.isAuthorized(request);
    return decision === "Allow";
  }
}
```

上記の`cedar.Authorizer`は架空のインターフェースですが、実際にはCedar SDKのメソッドでpolicyセットをロードし、`evaluate`または`isAuthorized`のような関数で判定を得ます。環境によって処理を切り替えられるようにしておけば、テスト時にはローカルエンジン、本番ではAVP、という使い分けも可能になります。

## まとめ: アーキテクチャの構成図 (Summary and Architecture)
本提案の権限管理アーキテクチャを図示すると、以下のような構成になります。

- **ユーザー/プロジェクト/タスク情報**はPostgreSQLで一元管理されます。アプリはここからユーザーの所属ロールやグループを取得できます。
- **Cedarポリシー**はAmazon Verified Permissionsのストアで集中管理され、デプロイ時や運用時イベントで更新されます。
- **バックエンド (Hono on Lambda)**は各APIリクエスト毎に認証→認可ミドルウェアを通し、AVPにクエリして許可判定を受けます。結果に応じて実処理を実行します。
- **管理者インターフェース**（例えばWeb管理画面）からグループ権限変更などが行われると、バックエンド経由でAVPのポリシーが更新されます。同時にPostgreSQLにも必要なメタ情報（どのグループにどの権限があるか等）を記録しておきます。
- **ログ/監査**: すべての認可チェック結果や更新操作は監査ログとして残されます。AVP自体もアクセス履歴を記録できるので、誰がどのポリシーを変えたか、どの要求が拒否されたか等を後から分析可能です。

このように、**ポリシーベース認可**を取り入れたZirconの権限管理アーキテクチャは、要件に沿った柔軟性と拡張性を提供します。新たなプロジェクトやグループが追加されてもコーディングレスで対応でき、セキュアで一貫したアクセス制御が可能となります。Cedarの導入によって将来的な要件変化（例えば権限条件の細分化や外部サービス連携による動的属性評価など）にも比較的容易に対応できる基盤が整ったと言えるでしょう。
