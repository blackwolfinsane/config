Application Runtime Configuration Extension for Yii 2
=====================================================

This extension provides support for application runtime configuration, loading config from database.

For license information check the [LICENSE](LICENSE.md)-file.

[![Latest Stable Version](https://poser.pugx.org/yii2tech/config/v/stable.png)](https://packagist.org/packages/yii2tech/config)
[![Total Downloads](https://poser.pugx.org/yii2tech/config/downloads.png)](https://packagist.org/packages/yii2tech/config)
[![Build Status](https://travis-ci.org/yii2tech/config.svg?branch=master)](https://travis-ci.org/yii2tech/config)


Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist yii2tech/config
```

or add

```json
"yii2tech/config": "*"
```

to the require section of your composer.json.


Usage
-----

This extension allows reconfigure already created Yii application instance using config composed from external storage
like relational database, MongoDB and so on. It allows to reconfigure any application property, component or module.
Configuration is performed by [[\yii2tech\config\Manager]] component, which should be added to the application configuration.
For example:

```php
[
    'bootstrap' => [
        'configManager',
        // ...
    ],
    'components' => [
        'configManager' => [
            'class' => 'yii2tech\config\Manager',
            'items' => [
                'appName' => [
                    'path' => 'name',
                    'label' => 'Application Name',
                    'rules' => [
                        ['required']
                    ],
                ],
                'nullDisplay' => [
                    'path' => 'components.formatter.nullDisplay',
                    'label' => 'HTML representing not set value',
                    'rules' => [
                        ['required']
                    ],
                ],
            ],
        ],
        ...
    ],
];
```

[[\yii2tech\config\Manager]] implements [[\yii\base\BootstrapInterface]] interface, thus being placed under 'boostrap'
section it will apply runtime configuration during application bootstrap. You can apply config manually to the application
or any [[\yii\base\Module]] descendant, using following code:

```php
$configManager = Yii::$app->get('configManager');
$configManager->configure(Yii::$app);
```


## Configuration items specification <span id="configuration-items-specification"></span>

Application parts, which should be reconfigured are determined by [[\yii2tech\config\Manager::items]], which is a list
of [[\yii2tech\config\Item]]. Each configuration item determines the configuration path - a list of keys in application
configuration array, which leads to the target value. For example: path 'components.formatter.nullDisplay' (or
`['components', 'formatter', 'nullDisplay']`) points to the property 'nullDisplay' of [[\yii\i18n\Formatter]] component,
path 'name' points to [[\yii\base\Application::name]] and so on.

> Note: if no path is specified it will be considered as a key inside [[\yii\base\Module::params]] array, which matches
  configuration item id (name of key in [[\yii2tech\config\Manager::items]] array).

Configuration item may also have several properties, which supports creation of web interface for configuration setup.
These are:
 - 'label' - string, input label.
 - 'description' - string, configuration parameter description or input hint.
 - 'rules' - array, value validation rules.
 - 'inputOptions' - array, list of any other input options.

Here are some examples of item specifications:

```php
'appName' => [
    'path' => 'name',
    'label' => 'Application Name',
    'rules' => [
        ['required']
    ],
],
'nullDisplay' => [
    'path' => 'components.formatter.nullDisplay',
    'label' => 'HTML representing not set value',
    'rules' => [
        ['required']
    ],
],
'adminEmail' => [
    'label' => 'Admin email address',
    'rules' => [
        ['required'],
        ['email'],
    ],
],
'adminTheme' => [
    'label' => 'Admin email address',
    'path' => ['modules', 'admin', 'theme'],
    'rules' => [
        ['required'],
        ['in', 'range' => ['classic', 'bootstrap']],
    ],
    'inputOptions' => [
        'type' => 'dropDown',
        'items' => [
            'classic' => 'Classic',
            'bootstrap' => 'Twitter Bootstrap',
        ],
    ],
],
```

> Tip: since runtime configuration may consist of many items and their declaration may cost a lot of code, it can
  be moved into a separated file and specified by this file name.


## Configuration storage <span id="configuration-storage"></span>

Declared configuration items may be saved into persistent storage and then retrieved from it.
The actual item storage is determined via [[\yii2tech\config\Manager::storage]].

Following storages are available:
 - [[\yii2tech\config\StoragePhp]] - stores configuration inside PHP files
 - [[\yii2tech\config\StorageDb]] - stores configuration inside relational database
 - [[\yii2tech\config\StorageMongoDb]] - stores configuration inside MongoDB
 - [[\yii2tech\config\StorageActiveRecord]] - finds configuration using ActiveRecord

Please refer to the particular storage class for more details.


## Creating configuration web interface <span id="creating-configuration-web-interface"></span>

The most common use case for this extension is creating a web interface, which allows control of application
configuration in runtime.
[[\yii2tech\config\Manager]] serves not only for applying of the configuration - it also helps to create an
interface for configuration editing.

The web controller for configuration management may look like following:

```php
use yii\base\Model;
use yii\web\Controller;
use Yii;

class ConfigController extends Controller
{
    /**
     * Performs batch updated of application configuration records.
     */
    public function actionIndex()
    {
        /* @var $configManager \yii2tech\config\Manager */
        $configManager = Yii::$app->get('configManager');

        $models = $configManager->getItems();

        if (Model::loadMultiple($models, Yii::$app->request->post()) && Model::validateMultiple($models)) {
            $configManager->saveValues();
            Yii::$app->session->setFlash('success', 'Configuration updated.'); 
            return $this->refresh();
        }

        $this->render('index', [
            'models' => $models,
        ]);
    }

    /**
     * Restores default values for the application configuration.
     */
    public function actionDefault()
    {
        /* @var $configManager \yii2tech\config\Manager */
        $configManager = Yii::$app->get('configManager');
        $configManager->clearValues();
        Yii::$app->session->setFlash('success', 'Default values restored.'); 
        $this->redirect(array('index'));
    }
}
```

The main view file can be following:

```php
<?php
use yii\helpers\ArrayHelper;
use yii\helpers\Html;
use yii\widgets\ActiveForm;

/* @var $models yii2tech\config\Item[] */
?>
<?php $form = ActiveForm::begin(); ?>

<?php foreach ($models as $model): ?>
    <?php
    $field = $form->field($model, 'value');
    $inputType = ArrayHelper::remove($model->inputOptions, 'type');
    switch($inputType) {
        case 'checkbox':
            $field->checkbox();
            break;
        case 'textarea':
            $field->textarea();
            break;
        case 'dropDown':
            $field->dropDownList($model->inputOptions['items']);
            break;
    }
    echo $field;
    ?>
<?php endforeach;?>

<div class="form-group">
    <?= Html::a('Restore defaults', ['index'], ['class' => 'btn btn-danger', 'confirm' => 'Are you sure you want to restore default values?']); ?>
    &nbsp;
    <?= Html::submitButton('Save', ['class' => 'btn btn-primary']) ?>
</div>

<?php ActiveForm::end(); ?>
```