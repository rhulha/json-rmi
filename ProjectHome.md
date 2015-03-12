This project consists of **one** class implementing complete Java Remote Method Invocation features using JSON. Another class exists to generate Java code from a JRIDL ( JSON Remote Interface Description Language ).

# Getting Started #

First open [JSON\_RMI](JSON_RMI.md) and [JRIDL](JRIDL.md)
and copy the class texts. Paste them into new classes in your favorite
IDE. Adapt the package declaration if you need to.

Then write a jridl file.
You can get inspiration from:
[ExampleJRIDL](ExampleJRIDL.md)

# Generating Java code from a JRIDL #

```

public class ReadJRIDL {
        public static void main(String[] args) throws Exception {
                JRIDL j = new JRIDL( "MyService", "myservice.jridl");
                j.gen("src/gen/");
        }
}

```

# Starting a Server #

```
public class TestServer {
	public static void main(String[] args) throws IOException {
		ServerSocket ss = new ServerSocket(25001);
		System.out.println("JSON-RMI SERVER started");
		while (true) {
			new Thread(new JSON_RMI(ss.accept(), new MyServiceImpl())).start();
		}
	}
}
```


# Starting a Client #

```

public class TestClient {
	public static void main(String[] args) throws IllegalArgumentException, IOException {
		MyService msc = JSON_RMI.getClient(MyService.class);
		System.out.println( msc.status());
		System.out.println( msc.echo("Hello"));
                // when you are done, close the connection.
		JSON_RMI ih = (JSON_RMI) Proxy.getInvocationHandler(msc);
		ih.close();
	}
}

```