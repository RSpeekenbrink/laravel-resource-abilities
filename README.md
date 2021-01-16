# Add abilities to Laravel API resources

If you build a web app with a separate frontend and backend, all kinds of information has to be transferred between 
these two parts. Part of this are your routes, but how do you share resource permissions in the most convenient way? We 
use API resources for this. That way we can see for each resource what permissions the current user has for that 
resource. However, when just adding the `Gate::check()`, a lot of gates may be checked which are not needed for that 
route. Therefore, this package gives you precise control over which gates exactly need to be checked. Let's take a 
look at such a resource:

```php
use AgilePixels\ResourceAbilities\ProcessesAbilities;
use App\Policies\PostPolicy;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    use ProcessesAbilities;

    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'published_at' => $this->published_at,

            'abilities' => $this->abilities(PostPolicy::class)
        ];
    }
}
```

Our package will scan the `PostPolicy` for available methods and check whether the gate allows or denies the user a 
certain permission. As a result, the json output will look like this:

```json
{
  "data": {
    "id": 10,
    "title": "Corporis eum adipisci et cum nostrum.",
    "slug": "corporis-eum-adipisci-et-cum-nostrum",
    "published_at": "2020-06-06T10:49:01.000000Z",
    "abilities": {
      "view": true,
      "update": false,
      "delete": true
    }
  }
}
```

## Installation

### Installing the package

You can install this package via composer:

```bash
composer require agilepixels/laravel-resource-abilities
```

The package will automatically register a service provider.

Publishing the config file is optional:

```bash
php artisan vendor:publish --provider="AgilePixels\ResourceAbilities\ResourceAbilitiesServiceProvider" --tag="config"
```

This is the default content of the config file:

```php
return [

    /*
    |--------------------------------------------------------------------------
    | Serializer
    |--------------------------------------------------------------------------
    |
    | The serializer will be used for the conversion of abilities to their
    | array representation, when no serializer is explicitly defined for an
    | ability resource this serializer will be used.
    |
    */

    'serializer' => AgilePixels\ResourceAbilities\Serializers\AbilitySerializer::class,
];
```

### Resource and model setup

This package gives you precise control over which abilities should be checked when hydrating your API resources. 
Therefore, some extra methods have to be added to be added to your models. In your models, add the `HasAbilities` 
trait.

```php
use AgilePixels\ResourceAbilities\HasAbilities;

class Post extends Model
{
    use HasAbilities;
}
```

In your resources, add the `ProcessesAbilities` trait and a new key where the abilities will be stored:

```php
use AgilePixels\ResourceAbilities\ProcessesAbilities;

class PostResource extends JsonResource
{
    use ProcessesAbilities;

    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'published_at' => $this->published_at,
            
            'abilities' => $this->abilities(PostPolicy::class)
        ];
    }
}
```

## Usage

### Resources

Laravel provides two primary ways of authorizing actions: gates and policies. Therefore, the available gates and 
policies must be defined first in the API resource.

#### Gates

To add a gate check to your API resource, use the `abilities()` method provided in de `ProcessesAbilities` trait. The 
given ability will be check using the model which is available in the resource. As an example, in a `PostResource`, the
ability 'view' will be checked using the ability `view` and the available `Post` model.

```php
public function toArray($request): array
{
    return [
        'abilities' => $this->abilities('view')
    ];
}
```

#### Policies

To check the current model against an entire policy, provide a policy in the `abilities()` method. This will add all 
available methods from the policy to the API resource.

```php
public function toArray($request): array
{
    return [
        'abilities' => $this->abilities(PostPolicy::class)
    ];
}
```

#### Adding multiple gates/policies

It may be the case that you want to add a policy but also some other gates. 

[comment]: <> (TODO Closure)

### Defining the abilities to be checked

Now that your gates and policies are defined, you would normally have all the information available in the frontend. 
On the other hand, your application quickly checks far too many gates and policies without them being necessary for the 
specific route. As a result, it can happen that unnecessary queries are executed, making your application less 
efficient and less fast. That is why this package offers the possibility to specify exactly which gates must be 
checked.

In your controller you can load the abilities in two ways, namely on the model itself or in the query builder. 

#### Loading abilities on model level

If your route is working with just one model, you can add an ability using the `addAbility()` method provided in the
`HasAbilities` trait. Typically, your controller should look something like this:

```php
public function show(Post $post)
{
    $post->addAbility('update');

    return PostResource::make($post);
}
```

The package will now check for the `update` gate and include the result in the response. This might look like the 
following example:

```json
{
    "data": {
        "id": 10,
        "title": "Corporis eum adipisci et cum nostrum.",
        "slug": "corporis-eum-adipisci-et-cum-nostrum",
        "published_at": "2020-06-06T10:49:01.000000Z",
        "abilities": {
          "update": false
        }
    }
}
```

[comment]: <> (TODO add multiple abilities in one call)

#### Loading abilities one builder level

But what if you're getting a whole collection of models. Performance wise it is inefficient to run across the entire 
collection and add the right abilities everywhere using a map. That is why it is also possible to add the abilities at 
the builder level, so that the abilities are immediately hydrated during the loading of the models. In your controller,
this looks like this:

```php
public function index()
{
    return PostResource::collection(
        Post::query()->addAbility('update')->get()
    );
}
```

The package will now automatically add the abilities to all posts. This can also be used to load certain relationship 
abilities from the current resource. Eloquent can ["eager load"](https://laravel.com/docs/master/eloquent-relationships#eager-loading) 
relationships at the time you query the parent model. It's also possible to indicate via a closure exactly how the 
relationship should be loaded. That is why it is also the perfect place to provide the abilities for the relationship.

```php
public function index()
{
    return PostResource::collection(
        Post::query()
            ->with(['author' => fn (BelongsTo $belongsTo) => $belongsTo->addAbility('view')])
            ->addAbility('update')
            ->get()
    );
}
```

The json response from your controller will now check the `view` gate for the author relationship.

```json
{
    "data": [
        {
            "id": 1,
            "title": "Perspiciatis veritatis rerum voluptatem reprehenderit earum rerum quod.",
            "slug": "perspiciatis-veritatis-rerum-voluptatem-reprehenderit-earum-rerum-quod",
            "published_at": "2018-04-12T23:31:59.000000Z",
            "author": {
                "id": 1053,
                "name": "Gavin Waters",
                "abilities": {
                    "view": true
                }
            },
            "abilities": {
                update": true
            }
        },
        // More posts with authors
    ]
}
```

## Testing

``` bash
composer test
```

## License

The MIT License (MIT). Please see [License File](https://github.com/agilepixels/laravel-resource-abilities/blob/master/LICENSE.md) for more information.