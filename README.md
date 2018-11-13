# A package to translate Laravel eloquent Models easily
This package serves as the complete solution to handling translations in a laravel application for data stored in the database. Translations for every model are stored in a database table. By default, automatic translation is enabled. Automatic translation uses third party provider(s) to translate your database attributes, the default for now is Google Translate API. If you choose not to use a third party translation service, you can also manually set your translation for any model.

## Installation 
You can install easily via composer.
```
composer install paschaldev/eloquent-translate
```

The package will automatically register itslef for supported laravel versions, if not, you should ad this to your providers array in `config/app.php`

```
PaschalDev\EloquentTranslate\Providers\TranslateServiceProvider::class,
```

Then after that you can publish 

```
php artisan vendor:publish --provider="PaschalDev\EloquentTranslate\Providers\TranslateServiceProvider"
```
This will copy the configuration to your config path.

For Lumen, open your `bootstrap/app.php` and add this line to regitser a provider 
```
$app->register(PaschalDev\EloquentTranslate\Providers\TranslateServiceProvider::class);
```

And this line to setup configuration 
```
$app->configure('eloquent-translate');
```

Then run `php artisan migrate` to create the translations table. The default table name is `translations` but you can override this value in the configuration file.

## Setup 

In order to make a model translatable, you need to include the trait in your model and also set the `translateAttributes` property. This properties are the attributes you wish to translate in that particukar DB column.

```
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;
use PaschalDev\EloquentTranslate\Traits\TranslatorTrait;

class Post extends Model
{
    public $translateAttributes = [

        'title',
        'content'
    ];
}
```

Once you have this done on your model, any update or create action on your model will automatically create your translations if you have the `auto_translate` set to `true` in your configuration.

#### Resolving trait collisions 
This package uses the `getAttribute` method in your model, you might already be using that or a library you have is using that and you start getting errors during translations because multiple packages are trying to set the same methods on a model. 

If you don't have a package that uses the `getAttribute` property, you can skip this section.

In order to resolve this, you need to make use of the `insteadof` keyword in your traits. For my own scenario, I had this library `Metable` that already uses the `getAttribute` method so here is how to resolve this issue.

```
<?php

namespace App;

use Kodeine\Metable\Metable;

use Illuminate\Database\Eloquent\Model;
use PaschalDev\EloquentTranslate\Traits\TranslatorTrait;

class Category extends Model{

    use Metable, TranslatorTrait {

        TranslatorTrait::getAttribute insteadof Metable;
        Metable::getAttribute as getAttributeOverride;
    }

    public $translateColumns = [

        'name',
        'caption'
    ];
}
```

As you can see, using the `insteadof`, we can tell the model to use this library's `getAttribute` methos instead of `Metable` and then we use the `as` keyword to 'rename' the `getAttribute` on `Metable` to `getAttributeOverride` so that we can still have access to it.

Lastly, you need to add this method in your model and make sure it calls the other library's `getAttribute` overriden name you defined using `as` above. In our case, `getAttributeOverride`. We need this because this package will first call this method before running it's own. That way we have deferred this package ro run `getAttribute` last so it doesn't affect your app.

```
public function getAttributeOverrider($key)
{
    return $this->getAttributeOverride($key);
}
```

After that, you're good to go. 


## Usage 

### Setting Translation 

Please see the configuration file `eloquent-translate.php` after publishing for all possible options. 

Originally, this package registers an observer to listen to the `created` and `updated` event in your model then automatically translate the attributes if you have `auto_translate` set to `true` in your configuration using a third party provider, the default for now is Google Translate API. 

If you're going to be using Google Translate API, you need to get a Google API Key from Google Console Cloud, make sure you enable translation API then copy and paste this key in your configuration file's `google_api_key`.

Automatic translation will likely slow down your app becuase it has to connect to Google and depending on the number of locales you set for it to translate to, your app will drag and might become unresponsive.

In order to fix this, make sure `queue` is set to true and you can optionally specify a `queue_name` to use. Please see Laravel's documentation on Queue if you don't understand how queues work.

Define your locales in the configuration file and when the package will translate your attributes to each of the locales defined.

If you prefer not to use automatic translation, you can manually set translations individually for your models. There are two methods available.

```
public function setTranslation($attribute, $locale, $translation, $force = false)
```

The attribute name, make sure it is defined in your `$translateAttributes` array in your model. The `$locale` you want to set and the translation.

The `$force` argument if set to true will make you add `attribute` even if the attributes are not defined in your model file.

```
$model->setTranslation('name', 'fr', 'Bonjour')
```

You can also set all your translations for a model in multiple locales at a go.

```
public function setTranslations($attribute, $translations)
```

Use it like this. 

```
$model = App\Post::find(1);

$translations = [

    'fr' => 'Bonjour',
    'ig' => 'ụtụtụ ọma',
    'yo' => 'e kaaro'
];

$model->setTranslations('name', $translations);
```

In the code above, we are setting translations for the `name` attribute on the model assuming the value is `Goof Morning` to 3 locales at once: French, Igbo, Yoruba.

### Fetching Translation 

