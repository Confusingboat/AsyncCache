# AsyncCache
[![Build](http://thinkquickbuild.cloudapp.net:8080/app/rest/builds/buildType:AsyncCache_Build/statusIcon)]
(http://thinkquickbuild.cloudapp.net:8080/project.html?projectId=AsyncCache&tab=projectOverview)
[![NuGet version](https://badge.fury.io/nu/AsyncCache.svg)](https://badge.fury.io/nu/AsyncCache)

Async cache uses an "async lock" (a semaphore) to prevent competing/simultaneous calls to the data source.

## Usage
AsyncCache is not a singleton. If you want your cached key/values to be scoped as a singleton (also just generally speaking), I recommend using a dependency injector to instantiate AsyncCache.

The reason the cached key/values are scoped to the instance of AsyncCache is this way you can control the both the TimeSpan and scope of your key/values.

**Usage Example Note**: Don't use *.Result* from a task. Async functions should be called by async functions. This is just an example app to illustrate usage of AsyncCache. This example app is synchronous..so maybe a bad example but it should still illistrate the usage ok.

        using Cache;
        using System;
        using System.Threading.Tasks;
        
        namespace ConsoleApplication1
        {
            class Program
            {
                static void Main(string[] args)
                {
                    var random = new Random();
                    var cache = new AsyncCache();
        
                    Console.WriteLine("Not Cached:");
                    for (var i = 0; i < 5; i++)
                        Console.WriteLine(random.Next(0, 5));
        
                    Console.WriteLine("\nCached:");
                    for (var i = 0; i < 5; i++)
                        Console.WriteLine(cache.Get(
                            key: "random integer",
                            dataSource: () => Task.FromResult(random.Next(0, 5))).Result);
        
                    Console.ReadKey();
                }
            }
        }

**Example App Output:**

        Not Cached:
        4
        2
        2
        4
        0
        
        Cached:
        2
        2
        2
        2
        2

Here is example usage from unit tests (below). Did I mention how simple it is?

        [TestInitialize]
        public void Setup()
        {
            _cache = new AsyncCache();
        }

        [TestMethod]
        public async Task ShouldReturnResultFromDataSourceTask()
        {
            //Arrange
            //Act
            int result = await _cache.Get("some key", () => Task.FromResult(2));

            //Assert
            result.Should().Be(2);
        }
        
        [TestMethod]
        public async Task ShouldLockToPreventRaceCondition()
        {
            var tcs1 = new TaskCompletionSource<int>();
            var tcs2 = new TaskCompletionSource<int>();

            int? result1 = null;
            int? result2 = null;
            var get1 = _cache.Get(key: "key1", dataSource: () => tcs1.Task).ContinueWith(t => result1 = t.Result);
            var get2 = _cache.Get(key: "key1", dataSource: () => tcs2.Task).ContinueWith(t => result2 = t.Result);

            tcs1.SetResult(1);
            tcs2.SetResult(2);

            await Task.WhenAll(get1, get2);

            result1.Should().Be(1, "because this is the initial value inserted into the cache.");
            result2.Should().Be(1, "because the previous/parallel request should've already inserted 1");
        }
