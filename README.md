# Commit 1 Reflection notes

For this first milestone, I tried to build a simple TCP connection handler using Rust’s standard library. At first, I didn’t realize that `TcpListener::bind` actually makes a system call to the OS to reserve a port (in this case, 7878). It makes sense why it returns a `Result`, since it can fail if the port is already in use or if there aren’t enough permissions.

In the `handle_connection` function, I found the use of `BufReader` pretty helpful. Without it, reading from the stream would happen byte by byte, which would result in a lot of system calls. By using `BufReader`, the data is buffered first, so it’s more efficient. For parsing the HTTP request, the code uses an iterator chain. Since data from a TCP stream comes in as raw bytes, `.lines()` helps convert it into UTF-8 strings and also splits it by lines. Then `.map()` is used to unwrap each result, and `.take_while()` keeps reading until it encounters an empty line. It turns out this is exactly how the HTTP protocol signals that the request headers have ended. Finally, `.collect()` is used to execute the whole chain and store the result in a `Vec<String>`.

I also noticed that the code uses `.unwrap()` quite a lot just to keep things simple. But in a real application, this could be risky, so proper error handling with `match` or the `?` operator would be better. Also, the stream parameter needs to be marked as `mut` because reading from it changes its internal state.

---

# Commit 2 Reflection notes

![Commit 2 screen capture](/assets/images/commit2.png)  

At first, I thought we could just send the HTML content directly through the stream, but it turns out it’s not that simple. HTTP responses have a specific format that needs to be followed, starting with a status line like `HTTP/1.1 200 OK`, followed by headers, then a blank line `(\r\n\r\n)`, and only after that the actual body. Without this structure, the browser won’t be able to understand the response properly.

For reading the HTML file, the code uses `fs::read_to_string("hello.html")`, which I found quite convenient because it handles opening, reading, and closing the file in one go. It returns a `Result<String>`, which makes sense since file operations can fail for various reasons.

One thing that stood out to me was the role of the `Content-Length` header. I didn’t realize before that TCP itself doesn’t define clear message boundaries, so the browser actually doesn’t know when the response ends unless we tell it. By including `Content-Length`, we specify exactly how many bytes the browser should read. Without it, the browser might just keep waiting for more data. That’s why `contents.len()` needs to be calculated and added to the headers before sending the response.

---

# Commit 3 Reflection notes

![Commit 3 screen capture](/assets/images/commit3.png)  

In this milestone, I added request validation to differentiate between a valid request to `/` and any other path, returning the appropriate HTML file and status code. Before refactoring, the naive approach duplicated the file-reading and response-sending logic inside both the `if` and `else` blocks. This violates the DRY (Don't Repeat Yourself) principle. If we ever needed to change the response format, we would have to update the code in multiple places, which is highly inefficient.

To solve this, I refactored the code by separating the decision from the execution. Because `if` is an expression in Rust, I used it to simply evaluate and return a tuple containing the `(status_line, filename)`. The shared execution logic—reading the file, calculating its length, and writing to the stream—is now written only once below the conditional block. This makes the code much cleaner, concise, and easier to maintain since there is only one central place to execute the response.

---

# Commit 4 Reflection notes

In this milestone, I simulated a slow response by adding a `/sleep` endpoint that forces the server to pause for 10 seconds using `thread::sleep`. This exercise perfectly demonstrates the fundamental limitation of our current single-threaded server. Because the program processes everything sequentially on one thread, it becomes entirely occupied until a request is completely finished. The thread is simply blocked waiting for the sleep duration to end.

When opening `127.0.0.1:7878/sleep` and a normal `/` request simultaneously in two browser tabs, I observed that the normal request is forced to wait until the 10-second sleep finishes, even though it requires zero computation to handle. The second request is basically stuck in the OS's connection queue because the only available thread is busy. In a real-world scenario, this means one slow request (like a heavy database query) would freeze the entire server for all other users.