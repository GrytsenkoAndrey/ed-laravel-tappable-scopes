# ed-laravel-tappable-scopes
Elevate Your Laravel Eloquent Queries with Tappable Scopes


Typically, when using query scopes in Laravel, the simple way is to use the scope prefix on a method in the model, like the following:

```
class Posts 
{		
	public function scopePublished(Builder $query): void
	{
		$query->where('published_at', '<=', now());
	}
}

$publishedPosts = Posts::published()->get();
```

This works well, but it does make it harder for the IDE to handle unless you’re using something like the Laravel IDE Helper package.

To convert this into a tappable scope, we can do something like the following:

```
// app/Scopes/Published.php

class Published
{
	public function __invoke(Builder $query): void
	{
		$query->where('published_at', '<=', now());
	}
}

$publishedPosts = Posts::tap(new Published)->get();
Using the tappable scope changes the following:

// With regular query scope
$publishedPosts = Posts::published()->get();

// With tappable scope
$publishedPosts = Posts::tap(new Published)->get();
```

The top one looks nicer, however, the IDE will not be able to easily see what the published method does since it using the magic scope prefix, whereas with the tappable scope version, you can easily click into Published and see exactly what’s happening.

Also, using the tappable scope allows it to be easily reused. For example, if you had a Comment model, that also included a published_at column, then to get just published comments, you can use the same scope from before:

```
$comments = Comment::tap(new Published)->get();
```

Now, let’s take these scopes to the next level by adding custom parameters.

Using are same Post and Comment models, let’s assume both include a user_id field. To handle that with a tappable scope, create the following:

```
// app/Scopes/ByUser.php

class ByUser
{
    public function __construct(private readonly int $userId)
    {
    }

    public function __invoke(Builder $query): void
    {
        $query->where('user_id', $this->userId);
    }
}
```

With the new tappable scope, we can fetch posts and comments for a user shown below:

```
$userId = 1;

$posts = Post::tap(new ByUser($userId))->get();

$comments = Comment::tap(new ByUser($userId))->get();
```

The above examples are relatively simple, and maybe it’s easier to just use normal where methods for those, so maybe they are not the best cases for tappable scopes, but I wanted to use the simple examples as an introduction. Now let’s create a tappable scope for something a little more complex.

On our home page, we want to show the latest published posts with the author and comment count. This query could look like the following:

```
$posts = Post::query()
    ->with(['user', 'comments'])
    ->where('published_at', '<=', now())
    ->latest('published_at')
    ->limit(10)
    ->get();
```
    
This works, but the query is starting to get kind of large. We also fetch the entire user model for each post and all the comments, when really all we want is a name and count. Also, we are counting unpublished comments which we don’t want. So let’s adjust:

```
$posts = Post::query()
    ->select('posts.*')
    ->join('users', 'users.id', '=', 'posts.user_id')
    ->withAggregate('user', 'name')
    ->withCount(['comments' => fn (Builder $query) => $query->where('published_at', '>=', now())])
    ->where('published_at', '<=', now())
    ->latest('published_at')
    ->limit(10)
    ->get();
```
    
This gives us exactly what we want, an array of posts with the following structure:

```
[
    0 => [    
        'id' => 69,
        'user_id' => 360,
        'name' => '...',
        'body' => '...',
        'published_at' => '2024-04-20 03:18:37',
        'created_at' => '2024-04-21T18:44:24.000000Z',
        'updated_at' => '2024-04-21T18:44:24.000000Z',
        'user_name' => 'Janae Luettgen',
        'comments_count' => 2,
    ],
    1 => [...]
    ...
]
```

This is great, but now our query is pretty complex. Imagine different parts of our application need to show a limit of 5 posts instead of 10. Or maybe we want to only find a count of unpublished comments. Let’s create a tappable scope:

```
// app/Scopes/LatestPosts.php

class LatestPosts
{
    public function __construct(private readonly int $limit = 10, private readonly bool $publishedComments = true)
    {
    }
    
    public function __invoke(Builder $query): void
    {
        $query->select('posts.*')
            ->join('users', 'users.id', '=', 'posts.user_id')
            ->withAggregate('user', 'name')
            ->withCount([
                    'comments' => fn (Builder $query) => $query
                        ->when(
                            $this->publishedComments,
                            fn(Builder $query) => $query->where('published_at', '>=', now())
                        )
                ]
            )
            ->where('published_at', '<=', now())
            ->latest('published_at')
            ->limit($this->limit);
    }
}
```

Now, instead of having to copy and paste this large query wherever we need it, it is encapsulated in a single place and we can fetch our latest posts like below:

```
$latestPosts = Post::tap(new LatestPosts(limit: 10, publishedComments: true))->get();
```

