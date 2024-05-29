# swift-modern-concurrency

```
import SwiftUI
import Foundation

// MARK: Domains
public struct Domains: Decodable {
  public let data: [Domain]
}

// MARK: - Domain
public struct Domain: Decodable {
  public let attributes: Attributes
}

// MARK: - Attributes
public struct Attributes: Decodable {
  public let name: String
  public let description: String
  public let level: String
}

// MARK: Playlist
public actor Playlist {
  // MARK: Properties
  public let title: String
  public let author: String
  public private(set) var songs: [String]

  // MARK: Initialization
  public init(title: String, author: String, songs: [String]) {
    self.title = title
    self.author = author
    self.songs = songs
  }

  // MARK: Methods
  public func add(song: String) {
    songs.append(song)
  }

  public func remove(song: String) {
    guard !songs.isEmpty, let index = songs.firstIndex(of: song) else {
      return
    }

    songs.remove(at: index)
  }

  public func move(song: String, from playlist: Playlist) async {
    await playlist.remove(song: song)
    add(song: song)
  }

  public func move(song: String, to playlist: Playlist) async {
    await playlist.add(song: song)
    remove(song: song)
  }
}

let task = Task {
    print("1")
    
    let sum = (1...1000000).reduce(0, +)
    // this checks a bool flag Task.isCancelled
    try Task.checkCancellation()
    print("2  sum", sum)
}

print("finish ")
task.cancel()
task.isCancelled

//Suspending tasks
Task {
    print("starting task....")
    // try inducates that Task.sleep can fail, and await inducates that it will be preformed async
    try await Task.sleep(nanoseconds: 1_000_000_000)
    print("finishing the task...")
}

// adding this async work to a func
// func that throws must be marked as such, also the func it self does not support async work in its natural form, but the sleep does and we await for it's excution, therefor the func must be marked as async

func preformTask() async throws {
    print("starting task....")
    try await Task.sleep(nanoseconds: 1_000_000_000)
    print("finishing the task...")
}

// to call the func async we use try await or await if it doesn't throw in both the declaration and the func call and this pattern is ment to keep things consistant between the two
Task {
    try await preformTask()
}

func fetchDomains() async throws -> [Domain] {
    let url = URL(string: "https://api.raywenderlich.com/api/domains")!
    // this method is async in the url session api, so it's called with await, this will free the progrm to do other things while the operation completes.
    // and because it can thorw errors it's marked with try
    // this call returns a tuple containig data and response, we take the data and ignore the response
    let (data, _) = try await URLSession.shared.data(from: url)
    //we then decode the data that was returned from the url
    return try JSONDecoder().decode(Domains.self, from: data).data
}

Task {
    do {
        let domains = try await fetchDomains()
        for (index, domain) in domains.enumerated() {
            let attributes = domain.attributes
            print("\(index + 1) \(attributes.name): \(attributes.description) - \(attributes.level)")
        }
        
    } catch {
        print(error)
    }
}

func findTitle(url: URL) async throws -> String? {
    
    //url has a property called lines that returns an asynchronous sequance of strings for each file line
    //for try await returns the title as soon as it can find the correct tag in it
    for try await line in url.lines {
        if line.contains("<title>") {
            return line
        }
    }
    return nil
}

Task {
    if let title = try await findTitle(url: URL(string: "https://www.raywenderlich.com")!) {
        print(title)
    }
}

// read-only computed properties can also be async
// this adds a static async computed property
extension Domains {
    static var domains: [Domain] {
        // because fetch domains is async and can throw erros we mark the getter with async throws
        get async throws {
            try await fetchDomains()
        }
    }
}

Task {
    dump(try await Domains.domains   )
}


// read-only subscripts can also  be async
// similar to the read-only aysync property, except this allows you to asynchrounosly subscript into your type to get the element you want.
extension Domains {
    enum Error: Swift.Error {case outOfRange }
    
    static subscript(_ index: Int) -> String {
        get async throws {
            let domains = try await self.domains
            guard domains.indices.contains(index) else {
                throw Error.outOfRange
            }
            return domains[index].attributes.name
        }
    }
}

Task {
    print("Async subscript")
    dump(try await Domains[4])
}

let favPlaylist = Playlist(title: "Favorite Songs", author: "Auth", songs: ["In and out of love"])

let partyPlaylist = Playlist(title: "Praty", author: "Auth", songs: ["Hello"])

Task {
    await favPlaylist.move(song:"Hello" , from: partyPlaylist)
    await favPlaylist.move(song: "In and out of love", to: partyPlaylist)
    await print("Fav", favPlaylist.songs)
    await print("Party", partyPlaylist.songs)
}
```
