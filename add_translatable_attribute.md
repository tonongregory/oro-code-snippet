##### 1. Create migration script to add relationship
```php
namespace Foo\Bar\Migrations\Schema\v1_0;

use Doctrine\DBAL\DBALException;
use Doctrine\DBAL\Schema\Schema;
use Doctrine\DBAL\Schema\SchemaException;
use Oro\Bundle\EntityConfigBundle\Migration\RemoveFieldQuery;
use Oro\Bundle\EntityExtendBundle\EntityConfig\ExtendScope;
use Oro\Bundle\EntityExtendBundle\Migration\Extension\ExtendExtension;
use Oro\Bundle\EntityExtendBundle\Migration\Extension\ExtendExtensionAwareInterface;
use Oro\Bundle\LocaleBundle\Entity\LocalizedFallbackValue;
use Oro\Bundle\MigrationBundle\Migration\Migration;
use Oro\Bundle\MigrationBundle\Migration\QueryBag;
use Oro\Bundle\ProductBundle\Entity\Product;

class MyCustomMigration implements Migration, ExtendExtensionAwareInterface
{
    /**
     * @var ExtendExtension
     */
    private $extendExtension;

    /**
     * @param ExtendExtension $extendExtension
     */
    public function setExtendExtension(ExtendExtension $extendExtension)
    {
        $this->extendExtension = $extendExtension;
    }

    /**
     * @param Schema $schema
     * @param QueryBag $queries
     * @throws DBALException
     * @throws SchemaException
     */
    public function up(Schema $schema, QueryBag $queries)
    {
            /** 
	    * Table of the entity to add custom property
            * In this case we add attibute on Oro\Bundle\ProductBundle\Entity\Product entity
	    **/
            $tableName = $this->extendExtension->getTableNameByEntityClass(Product::class);
            /** Create translatable field/column */
            $targetTableName = $this->extendExtension->getTableNameByEntityClass(LocalizedFallbackValue::class);
            $targetTable = $schema->getTable($targetTableName);
            // Column names are used to show a title of target entity
            $targetTitleColumnNames = $targetTable->getPrimaryKeyColumns();
            // Column names are used to show detailed info about target entity
            $targetDetailedColumnNames = $targetTable->getPrimaryKeyColumns();
            // Column names are used to show target entity in a grid
            $targetGridColumnNames = $targetTable->getPrimaryKeyColumns();
            $this->extendExtension->addManyToManyRelation(
                $schema,
                $tableName,
                'myTranslatableAttribute',
                $targetTableName,
                $targetTitleColumnNames,
                $targetDetailedColumnNames,
                $targetGridColumnNames,
                [
                    'attribute' => ['is_attribute => true],
                    'extend' => [
                        'owner' => ExtendScope::OWNER_CUSTOM,
                        'without_default' => true,
                        'cascade' => ['all'],
                    ],
                    'form' => ['is_enabled' => true],
                    'view' => ['is_displayable' => false],
                    'importexport' => [
                        'excluded' => false,
                        'fallback_field' => 'string',
                    ],
                ]
            );
        }
    }
}
```

##### 2. Add custom for guesser to add right form type on managment console
If you try to add new property on another entity than Oro\Bundle\ProductBundle\Entity\Product (ex custom entity created using CRUD) the edit form show the property as association. But we want to edit this property like a translatable field (with translate icon ![](https://camo.githubusercontent.com/cda5d2862e4267b7528322ff37a19ccc63591dc4/68747470733a2f2f662e636c6f75642e6769746875622e636f6d2f6173736574732f3831313139392f323034383336362f61646335376563652d386134332d313165332d393865612d3031636462613764366136662e706e67))
For this we have to create a custom form guesser like this

```php
<?php
namespace Foo\Bar\Form\Guesser;

use Symfony\Component\Form\Guess\TypeGuess;
use Oro\Bundle\EntityBundle\Form\Guesser\AbstractFormGuesser;
use Oro\Bundle\EntityConfigBundle\Config\Id\FieldConfigId;
use Oro\Bundle\EntityConfigBundle\Provider\ConfigProvider;
use Oro\Bundle\LocaleBundle\Entity\LocalizedFallbackValue;
use Oro\Bundle\LocaleBundle\Form\Type\LocalizedFallbackValueCollectionType;

/**
 * Class LocalizedValueTypeGuesser
 */
class LocalizedFallbackValueTypeGuesser extends AbstractFormGuesser
{
    /** @var ConfigProvider */
    protected $formConfigProvider;

    /**_ @var ConfigProvider */
    protected $extendConfigProvider;

    /**
     * @param ConfigProvider  $formConfigProvider
     * @param ConfigProvider  $extendConfigProvider
     */
    public function __construct(ConfigProvider $formConfigProvider, ConfigProvider $extendConfigProvider)
    {
        $this->formConfigProvider   = $formConfigProvider;
        $this->extendConfigProvider = $extendConfigProvider;
    }

    /**
     * @param string $className
     * @param string $property
     * @return TypeGuess|null
     */
    public function guessType($className, $property)
    {
        if (!$this->extendConfigProvider->hasConfig($className, $property)) {
            return $this->createDefaultTypeGuess();
        }

        $formConfig = $this->formConfigProvider->getConfig($className, $property);
        if (!$formConfig->is('is_enabled')) {
            return $this->createDefaultTypeGuess();
        }

        /** @var FieldConfigId $fieldConfigId */
        $fieldConfigId = $formConfig->getId();
        $fieldName     = $fieldConfigId->getFieldName();
        $extendConfig  = $this->extendConfigProvider->getConfig($className, $fieldName);
        $targetEntity = $extendConfig->getValues()['target_entity'] ?? null;
        // If target_entity is instance LocalizedFallbackValue guess type LocalizedFallbackValueCollectionType
        if ($targetEntity === LocalizedFallbackValue::class) {
            return $this->createTypeGuess(LocalizedFallbackValueCollectionType::class, ['block' => 'general']);
        }
    }
}
```
Tag your service with 'form.type_guesser' tag (Do not forgive to load your .yml file)
```txt
services:
  Foo\Bar\Form\Guesser\LocalizedFallbackValueTypeGuesser:
    arguments:
      - '@oro_entity_config.provider.form'
      - '@oro_entity_config.provider.extend'
    tags:
      - { name: form.type_guesser, priority: 25 }
```

##### 3. Run oro:platform:update command
