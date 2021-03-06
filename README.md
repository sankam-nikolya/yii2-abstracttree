CodiVerum Yii2 Extension for hierarchy data
========================================
Extension contains abstract classes making it easy to implement tree-structured data in Yii2 Framework.
It has what's needed in hierarchical data:
  - multiple root nodes
  - selecting parent, children, siblings, ancestors, descendants (also by distance); easy for programmer and also for DBMS.
  - modifying trees like adding nodes, removing nodes and moving nodes (with or without subtree); simple (it's based on `id_parent` value changes - all the work is done by `AbstractNode` class)

I have found other solutions implementing tree structured data (like nested sets)
but I don't like using recursion or subqueries for frequent queries like selecting data.
I created my own way (I haven't found that kind of solution).

I hope you'll find it useful and easy.

This is my first open-source work so I'll be glad to hear any suggestions.
I'm sure this little library may be improved.

The basic design puts the data into two tables:

1. node:
```
id INT PRIMARY KEY auto_increment,
id_parent_node INT DEFAULT NULL,
node_level INT NOT NULL
```

2. node_ancestor:
```
id_node INT NOT NULL,
id_ancestor_node INT NOT NULL,
```

Proper foreign keys are also set.
Note that names may be easily customized (see migration section below)

Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist codiverum/yii2-abstracttree "dev-master"
```

or add

```
"codiverum/yii2-abstracttree": "dev-master"
```

to the require section of your `composer.json` file.


Usage
-----

Using this extension you can easily make:
  - customized migration 
  - ready to use ActiveRecord classes 

#### Migration ####
To make migration, you should create new migration:

```
yii migrate/create migration_name
```

Your migration should extend `codiverum\abstracttree\migrations\AbstractTreeBaseMigration` class, for example:

~~~php
use codiverum\abstracttree\migrations\AbstractTreeBaseMigration;
use yii\db\Schema;

class m150505_194334_category_tree extends AbstractTreeBaseMigration {

    public function up() {
        parent::up();
        $this->addForeignKey('fk_category_usercreated', 'category', 'id_user_created', '{{%user}}', 'id', 'SET NULL', 'CASCADE');
    }

    public function getExtraNodeTableColumns() {
        return [
            'id_user_created' => Schema::TYPE_INTEGER,
            'name' => Schema::TYPE_STRING . '(128) NOT NULL',
            'description' => Schema::TYPE_TEXT,
            'created_at' => Schema::TYPE_INTEGER,
            'updated_at' => Schema::TYPE_INTEGER
        ];
    }

    public function getNodeTableName() {
        return 'category';
    }
}
~~~

Then apply migration using command:
```
yii migrate
```

#### Models ####

Generate pivot table using Gii and you can leave it as is.
After that you can use Gii to generate model for the main table or write it on you're own. Just make sure
your class is extending `AbstractNode` class.
If you did - you can delete relations created for default columns (not in `getExtraNodeTablesColumns` migration method)
as they are already coverd in `AbstractNode` class.
You may want to add to the description of your class (use your model class name instead of `Category`):

~~~php
 * @property Category $parent
 * @property Category[] $ancestors
 * @property Category[] $descendants
 * @property Category[] $children
 * @property Category[] $siblings
~~~

For migration example above, `Category` class would look like this:

~~~php
namespace common\models;

use codiverum\abstracttree\components\interfaces\TreeNodeInterface;
use codiverum\abstracttree\models\AbstractNode;
use Yii;
use yii\db\ActiveQuery;
use common\models\User;

/**
 * This is the model class for table "category".
 *
 * @property integer $id
 * @property integer $id_parent_category
 * @property integer $id_user_created
 * @property string $name
 * @property string $description
 * @property integer $created_at
 * @property integer $updated_at
 *
 * @property User $userCreated
 * @property Category $parent
 * @property Category[] $ancestors
 * @property Category[] $descendants
 * @property Category[] $children
 * @property Category[] $siblings
 */
class Category extends AbstractNode implements TreeNodeInterface {

    public function beforeSave($insert) {
        if ($insert) {
            $this->id_user_created = Yii::$app->user->id;
        }
        return parent::beforeSave($insert);
    }

    /**
     * @inheritdoc
     */
    public static function tableName() {
        return 'category';
    }

    /**
     * @inheritdoc
     */
    public function rules() {
        return [
            [['id_parent_category', 'category_level', 'id_user_created', 'created_at', 'updated_at'], 'integer'],
            [['name', 'category_level'], 'required'],
            [['description'], 'string'],
            [['name'], 'string', 'max' => 128],
            [['name', 'id_parent_category'], 'unique', 'targetAttribute' => ['name', 'id_parent_category'], 'message' => 'The combination of Id Parent Category and Name has already been taken.']
        ];
    }

    /**
     * @inheritdoc
     */
    public function attributeLabels() {
        return [
            'id' => Yii::t('db', 'ID'),
            'id_parent_category' => Yii::t('db', 'Id Parent Category'),
            'id_user_created' => Yii::t('db', 'Id User Created'),
            'name' => Yii::t('db', 'Name'),
            'category_category_level' => Yii::t('db', 'Category Level'),
            'description' => Yii::t('db', 'Description'),
            'created_at' => Yii::t('db', 'Created At'),
            'updated_at' => Yii::t('db', 'Updated At'),
            'parent.name' => Yii::t('db', 'Parent'),
            'userCreated.display_name' => Yii::t('db', 'Author'),
            'category_level' => Yii::t('db', 'Level'),
            'parent' => Yii::t('db', 'Parent'),
            'userCreated' => Yii::t('db', 'Author'),
        ];
    }

    /**
     * @return ActiveQuery
     */
    public function getUserCreated() {
        return $this->hasOne(User::className(), ['id' => 'id_user_created']);
    }

}
~~~


That's it.
