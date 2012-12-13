UPGRADE FROM 2.1 to 2.2
=======================

### HttpFoundation

 * MongoDbSessionHandler のデフォルトフィールド名と timestamp タイプが変更されました。

   `sess_` プレフィックスはデフォルトフィールド名から削除されました。セッション ID はデフォルトでは `_id` フィールドに保持されます。セッション日付は `MongoTimestamp` の代わりに `MongoDate`に保持されます。`gc()` メソッドに依存する代わりに MongoDB 2.2 以上のTTLコレクションを利用することも可能になりました。

 * ストップウォッチ機能は HttpKernel\Debug から独立したコンポーネントに移動されました。

 * _method リクエストパラメータのサポートはデフォルトで無効になりました (有効にするには、Request::enableHttpMethodParameterOverride() をコールしてください) 。

#### 非推奨

 * `Request::splitHttpAcceptHeader()` は非推奨で、Symfony 2.3 で削除されます。

   リクエストの accept-* ヘッダをパースする便利なメソッド群を提供する `AcceptHeader` を使用するようにしてください。次の例を見てください:

   ```
   $accept = AcceptHeader::fromString($request->headers->get('Accept'));
   if ($accept->has('text/html') {
       $item = $accept->get('html');
       $charset = $item->getAttribute('charset', 'utf-8');
       $quality = $item->getQuality();
   }

   // 項目は降順ソートされる
   $accepts = AcceptHeader::fromString($request->headers->get('Accept'))->all();

   ```

### フォーム

  * PasswordType はデフォルトでトリムされなくなりました。

### ルーティング

 * RouteCollection は木構造ではなく、Routes のフラットな配列のように動作します。このため、RouteCollection を構築するためにPHPを使用する場合、親コレクションに追加する前に子コレクションにルートを追加するのを忘れないようにしてください (ルートを定義するのにYAML や XML を使用している場合は関係ありません) 。

   変更前:

   ```
   $rootCollection = new RouteCollection();
   $subCollection = new RouteCollection();
   $rootCollection->addCollection($subCollection);
   $subCollection->add('foo', new Route('/foo'));
   ```

   変更後:

   ```
   $rootCollection = new RouteCollection();
   $subCollection = new RouteCollection();
   $subCollection->add('foo', new Route('/foo'));
   $rootCollection->addCollection($subCollection);
   ```

   また、すべての階層で `addCollection` をコールする必要があります。正しいシーケンスは次のようになります (逆は不可です) 。

   ```
   $childCollection->->addCollection($grandchildCollection);
   $rootCollection->addCollection($childCollection);
   ```

 * `RouteCollection::getParent()` と `RouteCollection::getRoot()` メソッドは非推奨で、Symfony 2.3 で削除されます。
 * デフォルト値や必須項目の値、任意項目の値を追加するための `RouteCollection::addPrefix` メソッドをプレフィックスの追加なしで誤用することはサポートされなくなりました。このため、`addPrefix` を `addPrefix('', $defaultsArray, $requirementsArray, $optionsArray)` のように空のプレフィックスもしくは `/` のみ (both have no relevance) でコールする場合、新しく提供された代替メソッド `addDefaults($defaultsArray)`, `addRequirements($requirementsArray)`, もしくは、 `addOptions($optionsArray)` を使用する必要があります。
 * `RouteCollection::addPrefix()` の `$options` オプションは非推奨になりました。オプションを追加することとパスのプレフィックスを追加することは関係ないためです。RouteCollection内のすべての子ルートにオプションを追加したい場合は、`addOptions()` を使用してください。
 * `RouteCollection::getPrefix()` メソッドは非推奨になりました。コレクション内のすべてのルートはこのプレフィックスを持っており、必ずしも真ではないからです。コレクションの先頭において、ツリー構造はもはやありませんので、このメソッドも無用です。
 * `RouteCollection::addCollection(RouteCollection $collection)` は単一パラメータのみで使用するようにしてください。他のパラメータ `$prefix`, `$default`, `$requirements`, `$options` はまだ有効ですが非推奨になりました。代わりに、`addPrefix` メソッドはこのユースケースで使用するようにしてください。
   変更前: `$parentCollection->addCollection($collection, '/prefix', array(...), array(...))`
   変更後:
   ```
   $collection->addPrefix('/prefix', array(...), array(...));
   $parentCollection->addCollection($collection);
   ```

### バリデータ

 * インターフェース郡は `ConstraintViolation`, `ConstraintViolationList`, `GlobalExecutionContext`, `ExecutionContext` の各クラスのために作られました。もし、これらのクラスをタイプヒントとして追加する場合、これらのインターフェースをタイプヒントとして追加することを推奨します。

   変更前:

   ```
   use Symfony\Component\Validator\ExecutionContext;

   public function validateCustomLogic(ExecutionContext $context)
   ```

   変更後:

   ```
   use Symfony\Component\Validator\ExecutionContextInterface;

   public function validateCustomLogic(ExecutionContextInterface $context)
   ```

   `ConstraintValidatorInterface` のすべての実装について、この変更は `initialize` メソッドに対して必須です。

   変更前:

   ```
   use Symfony\Component\Validator\ConstraintValidatorInterface;
   use Symfony\Component\Validator\ExecutionContext;

   class MyValidator implements ConstraintValidatorInterface
   {
       public function initialize(ExecutionContext $context)
       {
           // ...
       }
   }
   ```

   変更後:

   ```
   use Symfony\Component\Validator\ConstraintValidatorInterface;
   use Symfony\Component\Validator\ExecutionContextInterface;

   class MyValidator implements ConstraintValidatorInterface
   {
       public function initialize(ExecutionContextInterface $context)
       {
           // ...
       }
   }
   ```

#### 非推奨

 * `ClassMetadataFactoryInterface` インターフェースは非推奨で、Symfony 2.3 で削除されます。代わりに `MetadataFactoryInterface` を実装するようにしてください。メソッド名が `getClassMetadata` から `getMetadataFor` に変更され、任意の値 (たとえばクラス名やオブジェクト、数値など) を受け取ります。独自の実装では、与えられた値のメタデータをサポートしていない場合 `NoSuchMetadataException` を投げるようにしてください。

   変更前:

   ```
   use Symfony\Component\Validator\Mapping\ClassMetadataFactoryInterface;

   class MyMetadataFactory implements ClassMetadataFactoryInterface
   {
       public function getClassMetadata($class)
       {
           // ...
       }
   }
   ```

   変更後:

   ```
   use Symfony\Component\Validator\MetadataFactoryInterface;
   use Symfony\Component\Validator\Exception\NoSuchMetadataException;

   class MyMetadataFactory implements MetadataFactoryInterface
   {
       public function getMetadataFor($value)
       {
           if (is_object($value)) {
               $value = get_class($value);
           }

           if (!is_string($value) || (!class_exists($value) && !interface_exists($value))) {
               throw new NoSuchMetadataException(...);
           }

           // ...
       }
   }
   ```

   `ValidatorInterface::getMetadataFactory()` の戻り値も `MetadataFactoryInterface` に変更されました。このメソッドの戻り値の `getClassMetadata` を呼び出す箇所を `getMetadataFor` に変更することを忘れないようにしてください。

   変更前:

   ```
   $metadataFactory = $validator->getMetadataFactory();
   $metadata = $metadataFactory->getClassMetadata('Vendor\MyClass');
   ```

   変更後:

   ```
   $metadataFactory = $validator->getMetadataFactory();
   $metadata = $metadataFactory->getMetadataFor('Vendor\MyClass');
   ```

 * `GraphWalker` クラスとアクセサ `ExecutionContext::getGraphWalker()`
   は非推奨で、Symfony 2.3 で削除されます。代わりに、`ExecutionContextInterface::validate()` と `ExecutionContextInterface::validateValue()` メソッドを使用するようにしてください。

   変更前:

   ```
   use Symfony\Component\Validator\ExecutionContext;

   public function validateCustomLogic(ExecutionContext $context)
   {
       if (/* ... */) {
           $path = $context->getPropertyPath();
           $group = $context->getGroup();

           if (!empty($path)) {
               $path .= '.';
           }

           $context->getGraphWalker()->walkReference($someObject, $group, $path . 'myProperty', false);
       }
   }
   ```

   変更後:

   ```
   use Symfony\Component\Validator\ExecutionContextInterface;

   public function validateCustomLogic(ExecutionContextInterface $context)
   {
       if (/* ... */) {
           $context->validate($someObject, 'myProperty');
       }
   }
   ```

 * `ExecutionContext::addViolationAtSubPath()` メソッドは非推奨で、Symfony 2.3 で削除されます。代わりに `addViolationAt()` を使用してください。

   変更前:

   ```
   use Symfony\Component\Validator\ExecutionContext;

   public function validateCustomLogic(ExecutionContext $context)
   {
       if (/* ... */) {
           $context->addViolationAtSubPath('myProperty', 'This value is invalid');
       }
   }
   ```

   変更後:

   ```
   use Symfony\Component\Validator\ExecutionContextInterface;

   public function validateCustomLogic(ExecutionContextInterface $context)
   {
       if (/* ... */) {
           $context->addViolationAt('myProperty', 'This value is invalid');
       }
   }
   ```

 * `ExecutionContext::getCurrentClass()`, `ExecutionContext::getCurrentProperty()`, `ExecutionContext::getCurrentValue()` メソッドは非推奨で、Symfony 2.3 で削除されます。 代わりに `getClassName()`, `getPropertyName()`, `getValue()` を使用してください。

   変更前:

   ```
   use Symfony\Component\Validator\ExecutionContext;

   public function validateCustomLogic(ExecutionContext $context)
   {
       $class = $context->getCurrentClass();
       $property = $context->getCurrentProperty();
       $value = $context->getCurrentValue();

       // ...
   }
   ```

   変更後:

   ```
   use Symfony\Component\Validator\ExecutionContextInterface;

   public function validateCustomLogic(ExecutionContextInterface $context)
   {
       $class = $context->getClassName();
       $property = $context->getPropertyName();
       $value = $context->getValue();

       // ...
   }
   ```
