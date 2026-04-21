# Commit 1 Reflection notes

For this first milestone, I tried to build a simple TCP connection handler using Rust’s standard library. At first, I didn’t realize that `TcpListener::bind` actually makes a system call to the OS to reserve a port (in this case, 7878). It makes sense why it returns a `Result`, since it can fail if the port is already in use or if there aren’t enough permissions.

In the `handle_connection` function, I found the use of `BufReader` pretty helpful. Without it, reading from the stream would happen byte by byte, which would result in a lot of system calls. By using `BufReader`, the data is buffered first, so it’s more efficient. For parsing the HTTP request, the code uses an iterator chain. Since data from a TCP stream comes in as raw bytes, `.lines()` helps convert it into UTF-8 strings and also splits it by lines. Then `.map()` is used to unwrap each result, and `.take_while()` keeps reading until it encounters an empty line. It turns out this is exactly how the HTTP protocol signals that the request headers have ended. Finally, `.collect()` is used to execute the whole chain and store the result in a `Vec<String>`.

I also noticed that the code uses `.unwrap()` quite a lot just to keep things simple. But in a real application, this could be risky, so proper error handling with `match` or the `?` operator would be better. Also, the stream parameter needs to be marked as `mut` because reading from it changes its internal state.

---

# Commit 2 Reflection notes

![Commit 2 screen capture](/assets/images/image.png)

At first, I thought we could just send the HTML content directly through the stream, but it turns out it’s not that simple. HTTP responses have a specific format that needs to be followed, starting with a status line like `HTTP/1.1 200 OK`, followed by headers, then a blank line `(\r\n\r\n)`, and only after that the actual body. Without this structure, the browser won’t be able to understand the response properly.

For reading the HTML file, the code uses `fs::read_to_string("hello.html")`, which I found quite convenient because it handles opening, reading, and closing the file in one go. It returns a `Result<String>`, which makes sense since file operations can fail for various reasons.

One thing that stood out to me was the role of the `Content-Length` header. I didn’t realize before that TCP itself doesn’t define clear message boundaries, so the browser actually doesn’t know when the response ends unless we tell it. By including `Content-Length`, we specify exactly how many bytes the browser should read. Without it, the browser might just keep waiting for more data. That’s why `contents.len()` needs to be calculated and added to the headers before sending the response.