UPGRADE FROM 2.0 to 2.1
=======================

### 全般

  * `assets_base_urls` と `base_urls` のマージ戦略が変更されました。

    ほとんどの設定ブロックと異なり、`assets_base_urls` の一連の値はマージされるのではなくお互いを上書きします。この動作は選択されたもので、開発者は一般的に環境ごとに基準となるURLを定義しているためです。ほとんどのプロジェクトでは設定を継承しようとしている(たとえば、`config_test.yml` で `config_dev.yml` をimportする) 、および/または、共通の基底となる設定(たとえば、`config.yml`)を共有していることを考えれば、マージすることで複数環境用の基底URLのセットを生成できます

  * 組み込みリスナーのプロパティが変更されました。
 
    ```
                                            2.0         2.1
        security.firewall   kernel.request  64          8
        locale listener     kernel.request  0           16
        router listener     early_request   255         n/a
                            request         0           32
   ```

### Doctrine

  * DoctrineBundle は Symfony リポジトリから Doctrine リポジトリに移動されました。
    このため、AppKernel.php 内にあるバンドルの名前空間を変更する必要があります。

    変更前: `new Symfony\Bundle\DoctrineBundle\DoctrineBundle()`
    変更後: `new Doctrine\Bundle\DoctrineBundle\DoctrineBundle()`

### HttpFoundation

  * ロケール管理機能は Session クラスから Request クラスに移動されました。

    ##### デフォルトロケールの設定

    変更前:

    ```
    framework:
        session:
            default_locale: fr
    ```

    変更後:

    ```
    framework:
        default_locale: fr
    ```

  * `Request` クラスの `getPathInfo()`, `getBaseUrl()`, `getBasePath()` の各メソッドは生の値を返します(以前はurdecodeされた値)。これらのメソッドをコールする場合、必要であれば `rawurldecode()` でチェックしたりラップしなければなりません。

    ##### Twigテンプレートでロケールを処理する

    変更前: `{{ app.request.session.locale }}` もしくは `{{ app.session.locale }}`

    変更後: `{{ app.request.locale }}`

    ##### PHPテンプレートでロケールを処理する

    変更前: `$view['session']->getLocale()`

    変更後: `$view['request']->getLocale()`

    ##### PHPコードでロケールを処理する

    変更前: `$session->getLocale()`

    変更後: `$request->getLocale()`

### HttpFoundation

 * ユーザーに対する現在のロケールはセッションに保存されません。

   リクエスト中にあるロケール値を操作するパラメータが `_locale` の場合、以下の様なリスナーを登録することで、従来の動作をシミュレートすることができます。

   ```
   namespace XXX;

   use Symfony\Component\HttpKernel\Event\GetResponseEvent;
   use Symfony\Component\HttpKernel\KernelEvents;
   use Symfony\Component\EventDispatcher\EventSubscriberInterface;

   class LocaleListener implements EventSubscriberInterface
   {
       private $defaultLocale;

       public function __construct($defaultLocale = 'en')
       {
           $this->defaultLocale = $defaultLocale;
       }

       public function onKernelRequest(GetResponseEvent $event)
       {
           $request = $event->getRequest();
           if (!$request->hasPreviousSession()) {
               return;
           }

           if ($locale = $request->attributes->get('_locale')) {
               $request->getSession()->set('_locale', $locale);
           } else {
               $request->setLocale($request->getSession()->get('_locale', $this->defaultLocale));
           }
       }

       static public function getSubscribedEvents()
       {
           return array(
               // must be registered before the default Locale listener
               KernelEvents::REQUEST => array(array('onKernelRequest', 17)),
           );
       }
   }
   ```

### Security

  * `Symfony\Component\Security\Core\User\UserInterface::equals()` は `Symfony\Component\Security\Core\User\EquatableInterface::isEqualTo()` に移動されました。

    `User` クラスの実装に含まれる `equals()` を `isEqualTo()` に改名し、`EquatableInterface` を実装する必要があります。これ以外、変更はありません。

    独自の比較ロジックが必要ないのであれば、代わりに `AbstractToken::hasUserChanged()` が提供するデフォルトの実装を利用できます。この場合、`EquatableInterface` を実装せず、比較用のメソッドを削除してください。

    変更前:

    ```
    class User implements UserInterface
    {
        // ...
        public function equals(UserInterface $user) { /* ... */ }
        // ...
    }
    ```

    変更後:

    ```
    class User implements UserInterface, EquatableInterface
    {
        // ...
        public function isEqualTo(UserInterface $user) { /* ... */ }
        // ...
    }
    ```

  * ファイアウォール設定のための独自ファクトリは、エンドユーザーによって登録されるのではなく、バンドルの構築中に登録されるようになりました。このため、セキュリティ設定中の 'factories' キーを削除する必要があります。

  * FirewallリスナーはRouterリスナーの後に登録されます。この結果、特定のファイアウォールURL (/login_check や /logout など)は正確なルートをルーティング設定に記述しなければなりません。

  * ユーザープロバイダ設定がリファクタリングされ、プロバイダのチェインやメモリプロバイダの設定が変更されました。

     変更前:

     ``` yaml
     security:
         providers:
             my_chain_provider:
                 providers: [my_memory_provider, my_doctrine_provider]
             my_memory_provider:
                 users:
                     toto: { password: foobar, roles: [ROLE_USER] }
                     foo: { password: bar, roles: [ROLE_USER, ROLE_ADMIN] }
     ```

     変更後:

     ``` yaml
     security:
         providers:
             my_chain_provider:
                 chain:
                     providers: [my_memory_provider, my_doctrine_provider]
             my_memory_provider:
                 memory:
                     users:
                         toto: { password: foobar, roles: [ROLE_USER] }
                         foo: { password: bar, roles: [ROLE_USER, ROLE_ADMIN] }
     ```

  * `MutableAclInterface::setParentAcl` が `null` を受け付けるようになりました。この変更による影響があるかどうか、このインターフェースの実装を見なおしてください。

  * `UserPassword` 制約がSecurityバンドルからSecurityコンポーネントに移動されました。

     変更前:

     ```
     use Symfony\Bundle\SecurityBundle\Validator\Constraint\UserPassword;
     use Symfony\Bundle\SecurityBundle\Validator\Constraint as SecurityAssert;
     ```
     
     変更後:
     
     ```
     use Symfony\Component\Security\Core\Validator\Constraint\UserPassword;
     use Symfony\Component\Security\Core\Validator\Constraint as SecurityAssert;
     ```

### Form

#### FormTypeとオプションの後方互換性の欠落

  * `FormTypeInterface` と `FormTypeExtensionInterface` の `buildView()` と `buildViewBottomUp()` メソッドに第3引数 `$options` が追加されました。さらに、`buildViewBottomUp()` は `finishView()` に改名されました。これらのタイプのすべてのメソッドで、ようやく `FormBuilderInterface` のインスタンスを受け取るようになりました。以前は `FormBuilder` のインスタンスを受け取っていた部分です。以下のように、フォームタイプと拡張にあるメソッドシグネチャを変更する必要があります。

    変更前:

    ```
    use Symfony\Component\Form\FormBuilder;

    public function buildForm(FormBuilder $builder, array $options)
    ```

    変更後:

    ```
    use Symfony\Component\Form\FormBuilderInterface;

    public function buildForm(FormBuilderInterface $builder, array $options)
    ```

  * `createBuilder` メソッドはパフォーマンス的な理由により `FormTypeInterface` から削除されました。このため、特定のフォームタイプための `FormBuilderInterface` の独自実装が利用できなくなっています。

    もし、こういう状況になった場合、代わりに `FormRegistry` のサブクラスを作成し、独自の `ResolvedFormTypeInterface` 実装を返すよう `resolveType` をオーバーライドすることで、自身の `FormBuilderInterface` 実装を作成できます。そして、デフォルト実装を置き換えるために、この独自レジストリクラスをサービス名"form.registry"として登録してください。

  * `FieldType` を継承している場合、`FormType` を継承すべきです。そのフィールドが子フィールドを含まないハズなら、`compound` オプションを `false` にセットします。

    `FieldType` は非推奨で、Symfony 2.3 で削除されます。

    変更前:

    ```
    public function getParent(array $options)
    {
        return 'field';
    }
    ```

    変更後:

    ```
    public function getParent()
    {
        return 'form';
    }

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'compound' => false,
        ));
    }
    ```

    `getParent()` のシグネチャの変更については、次で説明します。
    新しいメソッド `setDefaultOptions` については、"非推奨" の節で説明します。

  * `FormTypeInterface` の `getParent()` にはオプションは渡せません。もし、`FormType` か `FieldType` を動的に継承している場合、代わりに "compound" オプションを動的にセットしてください。

    変更前:

    ```
    public function getParent(array $options)
    {
        return $options['expanded'] ? 'form' : 'field';
    }
    ```

    変更後:

    ```
    use Symfony\Component\OptionsResolver\OptionsResolverInterface;
    use Symfony\Component\OptionsResolver\Options;

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $compound = function (Options $options) {
            return $options['expanded'];
        };

        $resolver->setDefaults(array(
            'compound' => $compound,
        ));
    }

    public function getParent()
    {
        return 'form';
    }
    ```

    新しいメソッド `setDefaultOptions` については、"非推奨" の節で説明します。

  * フォームがオブジェクトにマップされている場合、"data_class" オプションをセットしなければなりません。もし空の場合、フォームは配列を期待するようになります。\ArrayAccess のインスタンス、もしくは、スカラー値であれば、それに対応する例外が発生します。

    同様に、フォームが配列もしくは \ArrayAccess のインスタンスにマップされている場合、このオプションは null のままにしておかなければなりません。

    フォームが `Person` インスタンスにマップされている例:

    ```
    use Symfony\Component\OptionsResolver\OptionsResolverInterface;

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'Acme\Demo\Person',
        ));
    }
    ```

  * プロパティパスから配列へのマッピングが変更されました。

    以前は、プロパティパス "street" はクラスのフィールド `$street` (もしくは、`getStreet()` と `setStreet()` というアクセッサ)と配列のインデックス `['street']` もしくは、`\ArrayAccess` を実装したオブジェクトの両方にマップされていました。

    現在では、プロパティパス "street" はクラスのフィールド(もしくはアクセッサ)のみにマップされます。"[street]"というプロパティパスのみインデックスにマップされます。

    もし、"property_path" オプションを使って手動でプロパティパスを定義した場合、必要に応じて修正や調整をしてください。

    変更前:

    ```
    $builder->add('name', 'text', array(
        'property_path' => 'address.street',
    ));
    ```

    変更後 (addressが配列の場合):

    ```
    $builder->add('name', 'text', array(
        'property_path' => 'address[street]',
    ));
    ```

    この場合、addressがオブジェクトであれば、"変更前"にあるコードは変更なしで動作します。
    works without changes.

  * フォームとフィールド名はレター、数字もしくはアンダースコアで始まり、レター、数字、アンダースコア、ハイフン、コロンで構成される必要があります。

  * コレクションタイプのテンプレートでは、プロトタイプフィールドのデフォルト名は `$$name$$` から `__name__` に変更されました。

    独自名にドル記号は戦闘や末尾に追加することはできません。フィールドの "prototype_name" に指定する値には、2つのアンダースコアを前後に追加するようにしてください。

    変更前:

    ```
    $builder->add('tags', 'collection', array('prototype' => 'proto'));

    // テンプレートでは "$$proto$$" という名前になる
    ```

    変更後:

    ```
    $builder->add('tags', 'collection', array('prototype' => '__proto__'));

    // テンプレートでは "__proto__" という名前になる
    ```

  * "read_only" オプションは `readonly="readonly"` と出力されます。`disabled="disabled"` としたい場合は "disabled" オプションを使用してください

  * 子フォームは自動的にバリデートされません。子フォームによって変更されたオブジェクトをバリデートしたいなら、明示的に `Valid` 制約をセットしてください。

    `Valid` 制約をセットしたくない、もしくは親フォームのデータから子フォームのデータを参照していない場合は、親フォームの "cascade_validation" オプションを `true` にすることで従来の動作をさせることができます。

#### テーマとHTMLにおける後方互換性の欠落

  * フォームタイプとフィールドタイプがマージされましたので、フォームテーマを適合させる必要があります。

    `field_widget` と関連するすべてのブロックは `form_widget_simple` に改名されました。

    変更前:

    ```
    {% block url_widget %}
    {% spaceless %}
        {% set type = type|default('url') %}
        {{ block('field_widget') }}
    {% endspaceless %}
    {% endblock url_widget %}
    ```

    変更後:

    ```
    {% block url_widget %}
    {% spaceless %}
        {% set type = type|default('url') %}
        {{ block('form_widget_simple') }}
    {% endspaceless %}
    {% endblock url_widget %}
    ```

    その他すべての `field_*` ブロックと関連するブロックは `form_*` に改名されました。事前に `field_*` と `form_*` ブロックの両方を定義している場合、代わりに、それらを1つの `form_*` ブロックにまとめ、新しい真偽値 `compound` を使ってチェックすることができます。

    変更前:

    ```
    {% block form_errors %}
    {% spaceless %}
        ... form code ...
    {% endspaceless %}
    {% endblock form_errors %}

    {% block field_errors %}
    {% spaceless %}
        ... field code ...
    {% endspaceless %}
    {% endblock field_errors %}
    ```

    変更後:

    ```
    {% block form_errors %}
    {% spaceless %}
        {% if compound %}
            ... form code ...
        {% else %}
            ... field code ...
        {% endif %}
    {% endspaceless %}
    {% endblock form_errors %}
    ```

    さらに、`generic_label` ブロックは `form_label` にマージされました。独自ラベルを出力するには、`form_label` をオーバーライドしてください。

    最後に、`widget_choice_options` ブロックは、その他のデフォルトテーマと一貫性を持たせるため、 `choice_widget_options` に改名されました。

  * choiceフィールドにおけるチェックボックスやラジオボタンの`id` や `name` のHTML属性の生成方法が変更されました。

    デフォルトでは、選択肢の値を追加する代わりに、生成された整数が追加されます。JavaScriptがこれに依存している場合は注意してください。実際の選択肢の値を参照したい場合は、代わりに `value` 属性を参照してください。

  * choiceフィールドタイプのテンプレートで、`choices` 変数の構成が変更されました。

    `choices` 変数は、選択肢のデータを参照するための `getValue()` と `getLabel()` という2つのゲッターを持つ `ChoiceView` オブジェクトを含みます。

    変更前:

    ```
    {% for choice, label in choices %}
        <option value="{{ choice }}"{% if _form_is_choice_selected(form, choice) %} selected="selected"{% endif %}>
            {{ label }}
        </option>
    {% endfor %}
    ```

    変更後:

    ```
    {% for choice in choices %}
        <option value="{{ choice.value }}"{% if _form_is_choice_selected(form, choice) %} selected="selected"{% endif %}>
            {{ choice.label }}
        </option>
    {% endfor %}
    ```

  * デフォルトラベルの生成はビューレイヤーに移動されました。`label` オプションが明示的にセットされていない場合に対応するには、独自の `form_label` テンプレートにこのロジックを組み込む必要があります。

    ```
    {% block form_label %}
        {% if label is empty %}
            {% set label = name|humanize %}
        {% endif %}

        {# ... #}

    {% endblock %}
    ````

  * コレクションフォームのそれぞれの行の独自スタイリングは、パフォーマンス的な理由により削除されました。代わりに、すべての行が、"entry" というワードが1つ前の行インデックスに置換される同じブロック名を持つようになります。

    変更前:

    ```
    {% block _author_tags_0_label %}
        {# ... #}
    {% endblock %}

    {% block _author_tags_1_label %}
        {# ... #}
    {% endblock %}
    ```

    変更後:

    ```
    {% block _author_tags_entry_label %}
        {# ... #}
    {% endblock %}
    ```

  * PHPテンプレーティングコンポーネント用ヘルパーのメソッド `renderBlock()` は、`block()` に改名されました。これの第1引数は `FormView` インスタンスです。

    変更前:

    ```
    <?php echo $view['form']->renderBlock('widget_attributes') ?>
    ```

    変更後:

    ```
    <?php echo $view['form']->block($form, 'widget_attributes') ?>
    ```

#### その他の後方互換性の欠落

  * `FormFactoryInterface` の `createNamed` と `createNamedBuilder` メソッドの先頭2引数の順序が、その他コンポーネントと一貫性を持たせるため、逆になりました。あなたのコードでこれらのメソッドが現れる箇所を調査し、パラメータを反転させてください。

    変更前:

    ```
    $form = $factory->createNamed('text', 'firstName');
    ```

    変更後:

    ```
    $form = $factory->createNamed('firstName', 'text');
    ```

  * `ChoiceList` の実装が大きく変更されました。この結果、`ArrayChoiceList` がリプレースされました。このクラスを拡張している独自クラスがある場合、`SimpleChoiceList` を拡張し、親コンストラクタに選択肢を渡してください。

    変更前:

    ```
    class MyChoiceList extends ArrayChoiceList
    {
        protected function load()
        {
            parent::load();

            // load choices

            $this->choices = $choices;
        }
    }
    ```

    変更後:

    ```
    class MyChoiceList extends SimpleChoiceList
    {
        public function __construct()
        {
            // load choices

            parent::__construct($choices);
        }
    }
    ```

    もし選択肢の読み込みを遅延させたい場合 -- 初回アクセス時に読み込む -- 代わりに `LazyChoiceList` を拡張し `loadChoiceList()` をオーバーライドして選択肢を読み込むことができます。

    ```
    class MyChoiceList extends LazyChoiceList
    {
        protected function loadChoiceList()
        {
            // load choices

            return new SimpleChoiceList($choices);
        }
    }
    ```

    `PaddedChoiceList`, `MonthChoiceList` そして `TimezoneChoiceList` は削除されました。これらは機能的に `DateType`, `TimeType` そして `TimezoneType` にマージされました。

    `EntityChoiceList` は適合されました。メソッド `getEntities()`,
    `getEntitiesByKeys()`, `getIdentifier()` そして `getIdentifierValues()` は削除、もしくはprivateメソッドに変更されました。最初の2つについては、代わりに `getChoices()` と `getChoicesByValues()` を利用できます。残りの2つについては、代替方法はありません。

  * HTML 属性は `form_label` 関数の `label_attr` 変数で渡すようになりました。

    変更前:

    ```
    {{ form_label(form.name, 'Your Name', { 'attr': {'class': 'foo'} }) }}
    ```

    変更後:

    ```
    {{ form_label(form.name, 'Your Name', { 'label_attr': {'class': 'foo'} }) }}
    ```

  * `EntitiesToArrayTransformer` と `EntityToIdTransformer` は削除されました。トランスフォーマは `CollectionToArrayTransformer` と `EntityChoiceList` の組み合わせに置き換えられ、後者はもはやコアに必要とされません。

  * 以下のトランスフォーマは、`ChoiceListInterface` での一貫性のため、改名されました。

      * `ArrayToBooleanChoicesTransformer` が `ChoicesToBooleanArrayTransformer` に
      * `ScalarToBooleanChoicesTransformer` が `ChoiceToBooleanArrayTransformer` に
      * `ArrayToChoicesTransformer` が `ChoicesToValuesTransformer` に
      * `ScalarToChoiceTransformer` が `ChoiceToValueTransformer` に

  * `FormUtil::toArrayKey()` と `FormUtil::toArrayKeys()` は削除されました。これらは、ChoiceList に統合され、これに相当するpublicなものはなくなりました。

  * フォームクラスの `add()`, `remove()`, `setParent()`, `bind()` そして `setData()` メソッドはフォームがすでにバインドされている場合に例外を投げるようになりました。

    バインドされているフォームにこれらのメソッドを使用した場合、`FormEvents::PRE_BIND` もしくは `FormEvents::BIND` を監視するイベントリスナーにロジックを移動することを検討してください。

#### 非推奨

  * `FormTypeInterface` と `FormTypeExtensionInterface` の以下のメソッドは非推奨で、Symfony 2.3 で削除されます。

      * `getDefaultOptions`
      * `getAllowedOptionValues`

    代わりに、新たに追加された `setDefaultOptions` を使用してください。これは、OptionsResolverInterface インターフェースにアクセスしたり多くのパワーを与えます。

    変更前:

    ```
    public function getDefaultOptions(array $options)
    {
        return array(
            'gender' => 'male',
        );
    }

    public function getAllowedOptionValues(array $options)
    {
        return array(
            'gender' => array('male', 'female'),
        );
    }
    ```

    変更後:

    ```
    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'gender' => 'male',
        ));

        $resolver->setAllowedValues(array(
            'gender' => array('male', 'female'),
        ));
    }
    ```

    クロージャを使ってその他のオプションに依存するオプションを指定できます。

    変更前:

    ```
    public function getDefaultOptions(array $options)
    {
        $defaultOptions = array();

        if ($options['multiple']) {
            $defaultOptions['empty_data'] = array();
        }

        return $defaultOptions;
    }
    ```

    変更後:

    ```
    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'empty_data' => function (Options $options, $value) {
                return $options['multiple'] ? array() : $value;
            }
        ));
    }
    ```

    第2引数 `$value` は現在のデフォルト値を含みます。必要でなければ指定する必要はありません。

  * `FormBuilder` の以下のメソッドは非推奨ですが、代替方法があります。

      * `prependClientTransformer`: `addViewTransformer`, 第2引数に `true` を指定する
      * `appendClientTransformer`: `addViewTransformer`
      * `getClientTransformers`: `getViewTransformers`
      * `resetClientTransformers`: `resetViewTransformers`
      * `prependNormTransformer`: `addModelTransformer`
      * `appendNormTransformer`: `addModelTransformer`, 第2引数に `true` を指定する
      * `getNormTransformers`: `getModelTransformers`
      * `resetNormTransformers`: `resetModelTransformers`

    非推奨となったメソッドは Symfony 2.3 で削除されますので、アプリケーションをアップデートすることを推奨します。

    変更前:

    ```
    $builder->appendClientTransformer(new MyTransformer());
    ```

    変更後:

    ```
    $builder->addViewTransformer(new MyTransformer());
    ```

  * 以下のイベントは非推奨となり、新しい代替方法があります。

      * `FormEvents::SET_DATA`: `FormEvents::PRE_SET_DATA`
      * `FormEvents::BIND_CLIENT_DATA`: `FormEvents::PRE_BIND`
      * `FormEvents::BIND_NORM_DATA`: `FormEvents::BIND`

    非推奨となったイベントは Symfony 2.3 で削除されます。

    さらに、イベントクラス `DataEvent` と `FilterDataEvent` は非推奨となり、一般的な `FormEvent` に置き換えられます。リスナーのコードを新しいイベントを使ったものに変更することを推奨します。非推奨となったイベントは Symfony 2.3 で削除されます。

    変更前:

    ```
    $builder->addListener(FormEvents::BIND_CLIENT_DATA, function (FilterDataEvent $event) {
        // ...
    });
    ```

    変更後:

    ```
    $builder->addListener(FormEvents::PRE_BIND, function (FormEvent $event) {
        // ...
    });
    ```

  * `FormValidatorInterface` インターフェースは非推奨となり、Symfony 2.3 で削除されます。

    このインターフェースを使用した独自バリデータを実装している場合、`FormEvents::POST_BIND` (もしくは、その他の `*BIND` イベント) をリッスンするイベントリスナーに置き換えることができます。CallbackValidator クラスを使っている場合は、`addEventListener` にコールバックを直接渡してください。

  * `FormTypeGuesserInterface` のメソッド `guessMinLength()` は非推奨となり、Symfony 2.3 で削除されます。HTML5の `pattern` 属性に挿入される正規表現を返す新しいメソッド `guessPattern()` を使用してください。

    変更前:

    ```
    public function guessMinLength($class, $property)
    {
        if (/* condition */) {
            return new ValueGuess($minLength, Guess::LOW_CONFIDENCE);
        }
    }
    ```

    変更後:

    ```
    public function guessPattern($class, $property)
    {
        if (/* condition */) {
            return new ValueGuess('.{' . $minLength . ',}', Guess::LOW_CONFIDENCE);
        }
    }
    ```

  * "property_path" を `false` にセットすることは推奨されません。Symfony 2.3 以降でサポートされなくなります。

    フィールドを親データとマッピングされないようにするには、新しいオプション "mapped" を使用してください。

    変更前:

    ```
    $builder->add('termsAccepted', 'checkbox', array(
        'property_path' => false,
    ));
    ```

    変更後:

    ```
    $builder->add('termsAccepted', 'checkbox', array(
        'mapped' => false,
    ));
    ```

  * `Form` の以下のメソッドは非推奨となり、Symfony 2.3 で削除されます。

      * `getTypes`
      * `getErrorBubbling`
      * `getNormTransformers`
      * `getClientTransformers`
      * `getAttribute`
      * `hasAttribute`
      * `getClientData`
      * `getChildren`
      * `hasChildren`
      * `bindRequest`

    変更前:

    ```
    $form->getErrorBubbling()
    ```

    変更後:

    ```
    $form->getConfig()->getErrorBubbling();
    ```

    メソッド `getClientData` は新しい等価な `getViewData` というメソッドになり、代わりに `FormConfigInterface` オブジェクトのその他のメソッドにアクセスできます。

    `getChildren` と `hasChildren` の代わりに、 `all` と `count` を使用してください。

    変更前:

    ```
    if ($form->hasChildren()) {
    ```

    変更後:

    ```
    if (count($form) > 0) {
    ```

    `bindRequest` の代わりに、シンプルに `bind` をコールしてください。

    変更前:

    ```
    $form->bindRequest($request);
    ```

    変更後:

    ```
    $form->bind($request);
    ```

  * オプション "validation_constraint" は非推奨となり、Symfony 2.3 で削除されます。フォームに1つ以上の制約を渡す場合、代わりに "constraints" オプションを使用してください。

    変更前:

    ```
    $builder->add('name', 'text', array(
        'validation_constraint' => new NotBlank(),
    ));
    ```

    変更後:

    ```
    $builder->add('name', 'text', array(
        'constraints' => new NotBlank(),
    ));
    ```

    前述のものとは異なり、制約のリストを渡すこともできます。

    ```
    $builder->add('name', 'text', array(
        'constraints' => array(
            new NotBlank(),
            new MinLength(3),
        ),
    ));
    ```

    制約は、属するバリデータグループでのみバリデートされます!なので、もしグループ "Custom" に属するフォームをバリデートする場合、それを明示しなければなりません。

    ```
    $builder->add('name', 'text', array(
        'validation_constraint' => new NotBlank(),
    ));
    ```

    この時、"Custom" グループに制約を追加する必要があります。

    ```
    $builder->add('name', 'text', array(
        'constraints' => new NotBlank(array('groups' => 'Custom')),
    ));
    ```

  * `DateType`, `DateTimeType` そして `TimeType` のオプション "data_timezone" と "user_timezone"は非推奨となり、Symfony 2.3 で削除されます。これらは "model_timezone" と "view_timezone" に改名されました。

    変更前:

    ```
    $builder->add('scheduledFor', 'date', array(
        'data_timezone' => 'UTC',
        'user_timezone' => 'America/New_York',
    ));
    ```

    変更後:

    ```
    $builder->add('scheduledFor', 'date', array(
        'model_timezone' => 'UTC',
        'view_timezone' => 'America/New_York',
    ));
    ```

  * `FormFactory` のメソッド `addType`, `hasType` そして `getType` は非推奨となり、Symfony 2.3 で削除されます。代わりに、`FormRegistry` にある同名のメソッドを使用してください。

    変更前:

    ```
    $this->get('form.factory')->addType(new MyFormType());
    ```

    変更後:

    ```
    $registry = $this->get('form.registry');

    $registry->addType($registry->resolveType(new MyFormType()));
    ```

  * `FormView` クラスの以下のメソッドは非推奨となり、Symfony 2.3 で削除されます。

      * `set`
      * `has`
      * `get`
      * `all`
      * `getVars`
      * `addChild`
      * `getChild`
      * `getChildren`
      * `removeChild`
      * `hasChild`
      * `hasChildren`
      * `getParent`
      * `hasParent`
      * `setParent`

    代わりに、publicなプロパティ `vars`, `children` そして `parent` にアクセスしてください。

    変更前:

    ```
    $view->set('help', 'A text longer than six characters');
    $view->set('error_class', 'max_length_error');
    ```

    変更後:

    ```
    $view->vars = array_replace($view->vars, array(
        'help'        => 'A text longer than six characters',
        'error_class' => 'max_length_error',
    ));
    ```

    変更前:

    ```
    echo $view->get('error_class');
    ```

    変更後:

    ```
    echo $view->vars['error_class'];
    ```

    変更前:

    ```
    if ($view->hasChildren()) { ...
    ```

    変更後:

    ```
    if (count($view->children)) { ...
    ```

### バリデータ

  * `ConstraintValidator` クラスの `setMessage()`, `getMessageTemplate()` そして `getMessageParameters()` メソッドは非推奨となり、Symfony 2.3 で削除されます。

    独自のバリデータを実装している場合、`ExecutionContext` オブジェクトの `addViolation()` メソッドを使用してください。

    変更前:

    ```
    public function isValid($value, Constraint $constraint)
    {
        // ...
        if (!$valid) {
            $this->setMessage($constraint->message, array(
                '{{ value }}' => $value,
            ));

            return false;
        }
    }
    ```

    変更後:

    ```
    public function isValid($value, Constraint $constraint)
    {
        // ...
        if (!$valid) {
            $this->context->addViolation($constraint->message, array(
                '{{ value }}' => $value,
            ));

            return false;
        }
    }
    ```

  * ExecutionContext クラスの `setPropertyPath()` は削除されました。

    代わりに、`ExecutionContext` オブジェクトの `addViolationAtSubPath()` メソッドを使用してください。

    変更前:

    ```
    public function isPropertyValid(ExecutionContext $context)
    {
        // ...
        $propertyPath = $context->getPropertyPath() . '.property';
        $context->setPropertyPath($propertyPath);
        $context->addViolation('Error Message', array(), null);
    }
    ```

    変更後:

    ```
    public function isPropertyValid(ExecutionContext $context)
    {
        // ...
        $context->addViolationAtSubPath('property', 'Error Message', array(), null);

    }
    ```

  * `ConstraintValidatorInterface` の `isValid` メソッドは `validate` に改名され、戻り値はなくなりました。

    `ConstraintValidator` にはまだ非推奨となった `isValid` メソッドがあり、デフォルトでは `validate` は `isValid` に転送されます。この後方互換レイヤーは Symfony 2.3 で削除されます。メソッドを改名することを推奨します。 また、戻り値を削除し、フレームワークで使われないようにしてください。

    変更前:

    ```
    public function isValid($value, Constraint $constraint)
    {
        // ...
        if (!$valid) {
            $this->context->addViolation($constraint->message, array(
                '{{ value }}' => $value,
            ));

            return false;
        }
    }
    ```

    変更後:

    ```
    public function validate($value, Constraint $constraint)
    {
        // ...
        if (!$valid) {
            $this->context->addViolation($constraint->message, array(
                '{{ value }}' => $value,
            ));

            return;
        }
    }
    ```

  * コア翻訳メッセージが変更されました。それぞれのメッセージの最後にドットが追加されました。上書きしたコア翻訳メッセージを修正する必要があります。

  * `Valid` アノテーションが付けられたプロパティ内のコレクション (配列もしくは `\Traversable` インスタンス) は、デフォルトでは再帰的にトラバースしません。

    コレクションであるエントリを含むコレクションの場合、内部のコレクションは以前のようにトラバースされません。`Valid` の新しいプロパティ `deep` を `true` にセットすることで、従来の動作をさせることができます。

    変更前:

    ```
    /** @Assert\Valid */
    private $recursiveCollection;
    ```

    変更後:

    ```
    /** @Assert\Valid(deep = true) */
    private $recursiveCollection;
    ```

  * `Size`, `Min` そして `Max` 制約は非推奨となり、Symfony 2.3 で削除されます。代わりに、新しい制約 `Range` を使用してください。

    変更前:

    ```
    /** @Assert\Size(min = 2, max = 16) */
    private $numberOfCpus;
    ```

    変更後:

    ```
    /** @Assert\Range(min = 2, max = 16) */
    private $numberOfCpus;
    ```

    変更前:

    ```
    /** @Assert\Min(2) */
    private $numberOfCpus;
    ```

    変更後:

    ```
    /** @Assert\Range(min = 2) */
    private $numberOfCpus;
    ```

  * `MinLength` と `MaxLength` 制約は非推奨となり、Symfony 2.3 で削除されます。代わりに、`Length` を使用してください。

    変更前:

    ```
    /** @Assert\MinLength(8) */
    private $password;
    ```

    変更後:

    ```
    /** @Assert\Length(min = 8) */
    private $password;
    ```

### セッション

  * フラッシュメッセージはタイプに基づく配列を返します。古いメソッドは利用可能ですが、非推奨です。

    ##### Twig テンプレートでフラッシュメッセージを処理する

    変更前:

    ```
    {% if app.session.hasFlash('notice') %}
        <div class="flash-notice">
            {{ app.session.getFlash('notice') }}
        </div>
    {% endif %}
    ```
    変更後:

    ```
    {% for flashMessage in app.session.flashbag.get('notice') %}
        <div class="flash-notice">
            {{ flashMessage }}
        </div>
    {% endfor %}
    ```

    1つのループですべてのフラッシュメッセージを処理することができます。

    ```
    {% for type, flashMessages in app.session.flashbag.all() %}
        {% for flashMessage in flashMessages %}
            <div class="flash-{{ type }}">
                {{ flashMessage }}
            </div>
        {% endfor %}
    {% endfor %}
    ```

  * セッションハンドラのドライバは `\SessionHandlerInterface` を実装する、もしくは
    `Symfony\Component\HttpFoundation\Session\Storage\Handler\NativeHandlerInterface` クラスを拡張し、`Handler\FooSessionHandler` という名前に変更します。たとえば、`PdoSessionStorage` は `Handler\PdoSessionHandler` となります。

  * `$session->*flash*()` メソッドを使用しているコードは、 `$session->getFlashBag()->*()` を使用するようリファクタリングしてください。

### シリアライザ

 * `GetSetMethodNormalizer` が生成するキー名が、すべて小文字のものからキャメルケースに変更されます(たとえば `mypropertyvalue` は `myPropertyValue` になります)。

 * `item` 要素は、XMLにデシリアライズされた場合に配列に変換されます。

    ``` xml
    <?xml version="1.0"?>
    <response>
        <item><title><![CDATA[title1]]></title></item><item><title><![CDATA[title2]]></title></item>
    </response>
    ```

    変更前:

        Array()

    変更後:

        Array(
            [item] => Array(
                [0] => Array(
                    [title] => title1
                )
                [1] => Array(
                    [title] => title2
                )
            )
        )

### ルーティング

  * UrlMatcher はルートパラメータを一度だけurldecodeします。以前は二度デコードしていました。複数回の `urldecode()` 呼び出しは入力パスの `+`  をサポートするために単一の `rawurldecode()` 呼び出しに変更されたことに注意してください。

  * DICに2つの新しいパラメータが追加されました。`router.request_context.host` と `router.request_context.scheme` です。機能テストや、cliコンテキスト時の正しいホストやスキームを用いたURL生成を行う場合にカスタマイズできます。

### FrameworkBundle

  * セッションのオプション: lifetime, path, domain, secure, httponly は非推奨となりました。
    代わりに、プレフィックスを付けたバージョンを使用してください: cookie_lifetime, cookie_path, cookie_domain, cookie_secure, cookie_httponly

  変更前:

  ```
    framework:
        session:
            lifetime:   3600
            path:       \
            domain:     example.com
            secure:     true
            httponly:   true
  ```

  変更後:

  ```
    framework:
        session:
            cookie_lifetime:   3600
            cookie_path:       \
            cookie_domain:     example.com
            cookie_secure:     true
            cookie_httponly:   true
  ```

`handler_id` が追加されました。デフォルトは `session.handler.native_file` です。

  ```
     framework:
         session:
             storage_id: session.storage.native
             handler_id: session.handler.native_file
  ```

モックのセッションストレージを使用する方法は以下のとおりとなります。`handler_id` はこのコンテキストにおいては意味がありません。

  ```
     framework:
         session:
             storage_id: session.storage.mock_file
  ```

### WebProfilerBundle

  * 2.1にアップグレードした後、古いプロファイルをクリアしなければなりません。データベースを使用している場合は、テーブルを削除する必要があります。
