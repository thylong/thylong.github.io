---
layout: post
title:  "Mocking Mongo in Golang"
description: "A usecase of interfaces to mock."
date:   2016-10-22 09:00:00
keywords: "golang, interface, mongodb, mongo, mock, testing, mgo"
category: golang
comments: true
---

<h2>Mocking Mongo in Golang</h2>

People coming from non-statically typed languages might be lost when it comes to mocking things in [Golang][golang].
This is especially true if you come to Go with a Python background and are used to using Python's wonderful [mock][mock_python] built-in library.
Although thanks to the power of interfaces and the great design of [Mgo][mgo] driver, it's relatively simple to Mock it.

<h3>Let's go</h3>
First, let's notice that the commonly used Mgo methods accept a pointer receiver and return a pointer receiver of the same package.

For example, look at these signatures :

> - func Dial(url string) (*Session, error)
> - func (s *Session) DB(name string) *Database
> - func (db *Database) C(name string) *Collection

This is really coherent as conventions say that if one method returns a pointer receiver, all of them should. But it doesn't really help us.

A solution to put it in place is a succession of interfaces and structs that will slightly modify the return values of the methods listed before.

To do that, let's start by abstracting the first struct we use from Mgo, the session :

{% highlight go %}
// Session is an interface to access to the Session struct.
type Session interface {
	DB(name string) DataLayer
	Close()
}

// MongoSession is currently a Mongo session.
type MongoSession struct {
	*mgo.Session
}

// DB shadows *mgo.DB to returns a DataLayer interface instead of *mgo.Database.
func (s MongoSession) DB(name string) DataLayer {
	return &MongoDatabase{Database: s.Session.DB(name)}
}

// DataLayer is an interface to access to the database struct.
type DataLayer interface {
	C(name string) Collection
}
{% endhighlight %}

We can implement the structs for the mock as follows :

{% highlight go %}
// MockSession satisfies Session and act as a mock of *mgo.session.
type MockSession struct{}

// NewMockSession mock NewSession.
func NewMockSession() Session {
	return MockSession{}
}

// Close mocks mgo.Session.Close().
func (fs MockSession) Close() {}

// DB mocks mgo.Session.DB().
func (fs MockSession) DB(name string) DataLayer {
	mockDatabase := MockDatabase{}
	return mockDatabase
}
{% endhighlight %}

Now it's totally possible to create/manipulate an interface :

{% highlight go %}
// NewSession returns a new Mongo Session.
func NewSession() Session {
	mgoSession, err := mgo.Dial("<MONGO_URI>")
    if err != nil {
        panic(err)
    }
	return MongoSession{mgoSession}
}

// OR

// NewMockSession mock NewSession.
func NewMockSession() Session {
	return MockSession{}
}
{% endhighlight %}

> N.B: In the case of mock session, no external calls are made.

We can then replicate that strategy with the database and the collection.
Note that for the sake of simplicity, the interfaces contain the bare minimum number of methods but if you plan to use more methods of the Collection or Database structs, you should add them to the interfaces.

{% highlight go %}
// MongoCollection wraps a mgo.Collection to embed methods in models.
type MongoCollection struct {
	*mgo.Collection
}

// Collection is an interface to access to the collection struct.
type Collection interface {
	Find(query interface{}) *mgo.Query
	Count() (n int, err error)
	Insert(docs ...interface{}) error
	Remove(selector interface{}) error
	Update(selector interface{}, update interface{}) error
}

// MongoDatabase wraps a mgo.Database to embed methods in models.
type MongoDatabase struct {
	*mgo.Database
}

// C shadows *mgo.DB to returns a DataLayer interface instead of *mgo.Database.
func (d MongoDatabase) C(name string) Collection {
	return &MongoCollection{Collection: d.Database.C(name)}
}

{% endhighlight %}

and for the mock :

{% highlight go %}
// MockDatabase satisfies DataLayer and act as a mock.
type MockDatabase struct{}

// MockCollection satisfies Collection and act as a mock.
type MockCollection struct{}

// Find mock.
func (fc MockCollection) Find(query interface{}) *mgo.Query {
	return nil
}

// Count mock.
func (fc MockCollection) Count() (n int, err error) {
	return 10, nil
}

// Insert mock.
func (fc MockCollection) Insert(docs ...interface{}) error {
	return nil
}

// Remove mock.
func (fc MockCollection) Remove(selector interface{}) error {
	return nil
}

// Update mock.
func (fc MockCollection) Update(selector interface{}, update interface{}) error {
	return nil
}

// C mocks mgo.Database(name).Collection(name).
func (db MockDatabase) C(name string) Collection {
	return MockCollection{}
}
{% endhighlight %}

All of our functions will then deal with interfaces instead of the underlying structs (*that can be either a real database or a mock*).

This solution is also great because it gives us an abstract layer. We can use the interfaces from the model in a handler, for example. The resulting codebase will be less coupled to the database solution we picked (*in this case Mongodb*) and easier to test.
Note that in this post, I give you only one of the possibilities to Mock, the one that made the most sense to me. You don't have to mock all the way from the session to the query structs. It really depends on your test strategy.
You can also use [GoMock][gomock] package even though interfaces do the job.

You can find an example of a real use case in [RegexRace][regexrace_repo] repository. The code shown above can also be found in [this repository][example_repo].
Feel free to ask me any questions in the comments or to open issues on Github.

If you're interested in Golang testing techniques, I suggest that you watch the [great talk][great_talk] of [Mitchell Hashimoto][mitchellh] and if you enjoyed this article, don't hesitate to star the repository or share it ;)


Cheers !

[golang]:          http://www.unixstickers.com/image/cache/data/stickers/golang/golang.sh-600x600.png
[mgo]:             https://labix.org/mgo
[mock_python]:     https://docs.python.org/3/library/unittest.mock.html
[regexrace_repo]:  http://github.com/thylong/regexrace
[example_repo]:    http://github.com/thylong/mongo_mock_go_example
[gomock]:          https://github.com/golang/mock
[mitchellh]:       http://mitchellh.com/
[great_talk]:      https://www.youtube.com/watch?v=yszygk1cpEc
