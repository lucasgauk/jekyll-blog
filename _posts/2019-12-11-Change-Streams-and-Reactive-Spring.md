---
layout: post
title: Change Streams, Reactive Programming, and Nonsense
---
Today we're going to explore a bit of what we can do with a combination of super interesting Spring / MongoDB tools.
Namely, we're gonna get weird with MongoDB change streams and Spring WebFlux. 

Before we get started I'd like to mention that none of the code explored in this post is production ready.
This is just stuff that I find interesting.

A repository containing a working copy of the code can be found [here](https://github.com/lucasgauk/mongodb-change-stream).

## WebFlux

WebFlux is a huge Spring library that brings support for reactive programming to Spring Boot. 
Built on top of [reactor](https://projectreactor.io/), it exposes some super interesting stream tools 
that adhere to the [reactive-streams](http://www.reactive-streams.org/) specifications. Today we'll be looking 
at a few of these tools: `Mono` and `Flux`. Mono represents an asynchronous response of 0 or 1 items, where as a Flux
represents an asynchronous sequence of 0 to n items.

Let's start by setting up a simple reactive API.

First we'll need to add the following dependencies to our project

``` 
<dependency>                                            
    <groupId>org.springframework.boot</groupId>         
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

Then we'll create a few simple models to represent a very basic Post / Comment cycle

```java 
@Data
public class BaseEntity {

    @Id
    private ObjectId id;
    @CreatedDate
    private LocalDateTime createdAt;

}
```

```java 
@Getter
@Setter
@Document
@TypeAlias("Post")
@NoArgsConstructor
public class Post extends BaseEntity {

    private String content;
    private List<Comment> comments;

    private Post(String content) {
        this.content = content;
        this.comments = new ArrayList<>();
    }

    public static Post from(PostRequest request) {
        return new Post(request.getContent());
    }

}
```

```java 
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Comment {

    private String content;
    private LocalDateTime createdAt;

    public static Comment from(CommentRequest request) {
        return new Comment(request.getContent(), LocalDateTime.now());
    }

}
```

Next, and crucially, we'll create a new repository for our post collection

```java 
@Repository
public interface PostRepository extends ReactiveMongoRepository<Post, String> {
}
```

The `ReactiveMongoRepository` exposes a number of reactive methods for us
to use to access our collections. Let's finish up our API and see how they behave.

I'm going to wrap the repository in a service

```java 
@Service
public class PostServiceImpl implements PostService {

    private final PostRepository repository;

    public PostServiceImpl(PostRepository repository) {
        this.repository = repository;
    }

    public Mono<Post> save(Post post) {
        return this.repository.save(post);
    }

    public Flux<Post> findAll() {
        return this.repository.findAll();
    }

    public Mono<Post> addComment(Comment comment, String postId) {
        return this.find(postId).flatMap(post -> {
            post.getComments().add(comment);
            return this.save(post);
        });
    }

    public Mono<Post> find(String postId) {
        return this.repository.findById(postId);
    }
}
```

The only interesting thing here is the addComment method. Notice that we're doing things a bit differently.
We're returning the `find()` stream but adding a pipeline that takes the post we find, 
adds the comment, and saves it. It's a bit different than our usual non reactive style. If you have experience
with Angular and Observables this should be second nature.

And add a quick controller

```java 
@RestController
@RequestMapping("/post")
@CrossOrigin(origins = "*", maxAge = 3600)
public class PostController {

    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping
    public Flux<Post> findAll() {
        return this.postService.findAll();
    }

    @GetMapping("/{id}")
    public Mono<Post> find(@PathVariable String id) {
        return this.postService.find(id);
    }

    @PostMapping("/{id}/comment")
    public Mono<Post> comment(@RequestBody CommentRequest request, @PathVariable String id) {
        return this.postService.addComment(Comment.from(request), id);
    }

    @PostMapping("/create")
    public Mono<Post> create(@RequestBody PostRequest postRequest) {
        return this.postService.save(Post.from(postRequest));
    }
    
}
``` 

If you run this and ping the endpoints you'll notice that it behaves
exactly the same way that it would if we had created this API using
conventional non reactive methods. Although it behaves the same way 
to the client, Spring is handling things very differently.
 
Flux streams are true streams. You'll never notice it in our little
API, but we now have a throttled flow of data through our application.
In non reactive applications we have the concept of back-pressure,
where a stream producer can create data faster than the receiver can process
it creating a build up of buffered unhandled data. Our throttled 
streams will never allow that to happen. 
 
This is a super handy concept to keep in mind when you need to perform operations 
on huge amounts of data without using disastrous amounts of memory.
 
## Change Streams
 
Now that we've talked a bit about WebFlux let's have a look the MongoDB change stream.

Change streams are a super cool MongoDB feature that allow us to subscribe to changes happening
on our collections and react to them in real time. Change streams rely on the replication 
feature of MongoDB. Clustered databases are already forced to share changes to collections to all
members in the cluster. Change streams allow us to tap into this conversation.

In order for there to be this conversation in the first place, our MongoDB database needs to be set
up as a replica set. This is best done locally by using docker compose, but for anyone else running
their MongoDB as a service on Windows Home (terrible I know) here's a quick and dirty guide on how to 
set it up as a replica set with a single member. 

Find your mongod.cfg file. Add the following 
``` 
replication:
  replSetName: replocal
```

Restart MongoDB. You can use the windows service manager if it is installed as a service.
Connect using the mongo shell. Run the following command

``` 
rs.initiate({_id: "replocal", members: [{_id: 0, host: "127.0.0.1:27017"}] })
```

You should now have a super robust single member cluster.

Now this is a contrived use case but stay with me here. Let's add a fancy little method
in our PostService that allows our users to subscribe to a stream of any changes happening
on a Post.

```java 

private final ReactiveMongoTemplate reactiveTemplate;

// ...

public Flux<Post> subscribe(String postId) {
    Aggregation fluxAggregation = newAggregation(match(where("operationType")
                                                           .is("update")
                                                           .and("fullDocument._id")
                                                           .is(new ObjectId(postId))));

    ChangeStreamOptions options = ChangeStreamOptions.builder()
                                                     .returnFullDocumentOnUpdate()
                                                     .filter(fluxAggregation)
                                                     .build();

    return reactiveTemplate.changeStream("post",
                                         options,
                                         Post.class)
                           .map(ChangeStreamEvent::getBody);
}
``` 

Notice I'm able to add an Aggregation operation to filter fields on the document that
has been updated and the operationType of the update. The information that comes out
of the change stream is honestly unbelievably useful. 

Take a minute to let this little method sink in. Any time the Post matching the ID you
have passed gets updated, you're going to receive an item in your async stream. The use
cases for this are insane. Imagine a microservice architecture using a messaging bus. You could
use this stream to trigger updates. You could also store the changed documents to trace back problematic
changes in your data. Let's combine what we looked at with WebFlux with these new change stream tools 
and look at one more potential use case.

With any stream, nothing is going to happen unless someone subscribes. So lets give the people
what they want. In our controller let's add the following

```java 
@GetMapping(path = "/{id}/subscribe", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Post> subscribe(@PathVariable String id) {
    return this.postService.subscribe(id);
}
```

Pick a post and open this endpoint in your browser (not IE) by going to the URL. You'll notice this behaves 
a bit differently than the other Flux endpoints from before. You'll notice your browser 
hangs on this endpoint. Try going into your database and changing something in the Post
that you've subscribed to. You just got a new asynchronous message baby. 

I've gotta say that I am unreasonably excited about this. We have an endpoint that 
exposes an asynchronous event driven stream of ANY changes that happen in our database.

## Conclusion
Today we've gotten a little introduction on a few reactive tools in Spring and some
potential use cases for them. 

As I mentioned before, careful when using this code. This is all fairly new stuff so I'm
not super confident with conventions, etc. And stay tuned for a post where I create a little Angular application
that uses our new event driven endpoint to display Post updates in real time.

Have fun and happy coding!


 


