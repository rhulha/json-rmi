#labels JSON,RPC,RMI

```Java

package org.json.rmi;

/*
 * License: http://www.apache.org/licenses/LICENSE-2.0.html
 * Author: Raymond Hulha
 */

import java.io.*;
import java.lang.reflect.*;
import java.net.Socket;
import java.nio.charset.Charset;
import java.util.*;
import java.util.Map.Entry;

public class JSON_RMI implements InvocationHandler, Runnable {
	
	Charset utf8 = Charset.forName("utf-8");
	
	private static final int port = 25001;

	private Socket socket;
	private Map<String, Method> methodMap = new HashMap<String, Method>();
	private Object service;

	public JSON_RMI() {
		// client
	}

	public JSON_RMI(Socket socket, Object service) {
		// server
		this.socket = socket;
		this.service = service;
		for (Method method : service.getClass().getDeclaredMethods()) {
			methodMap.put(method.getName(), method);
		}
	}

	@SuppressWarnings("unchecked")
	public static <T> T getClient(Class<T> t) throws IllegalArgumentException, IOException {
		return (T) Proxy.newProxyInstance(JSON_RMI.class.getClassLoader(), new Class<?>[] { t }, new JSON_RMI());
	}

	// SERVER

	@Override
	public void run() {
		try {
			BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream(), utf8));
			BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream(), utf8));

			while (true) {
				Object parseResult = parse(br);
				if (parseResult == null)
					break;
				serve(parseResult, bw);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	private void serve(Object parseResult, BufferedWriter bw) throws IOException, IllegalAccessException, InvocationTargetException, InstantiationException {
		@SuppressWarnings("unchecked")
		Map<String, Object> methodInvocations = (Map<String, Object>) parseResult;
		Set<Entry<String, Object>> methodInvocationEntrySet = methodInvocations.entrySet(); // should only be one...
		for (Entry<String, Object> methodInvocation : methodInvocationEntrySet) {
			Method method = methodMap.get(methodInvocation.getKey());

			@SuppressWarnings("unchecked")
			List<Object> params = (List<Object>) methodInvocation.getValue();

			List<Object> realParams = new ArrayList<Object>();
			Class<?>[] parameterTypes = method.getParameterTypes();

			for (int i = 0; i < parameterTypes.length; i++) {
				Class<?> type = parameterTypes[i];

				Object param = params.get(i);
				if (param instanceof String) {
					realParams.add(param);
				} else {
					@SuppressWarnings("unchecked")
					HashMap<String, Object> p = (HashMap<String, Object>) params.get(i);

					realParams.add(deSerialize(type.newInstance(), p));
				}
			}
			Object result = method.invoke(service, realParams.toArray());
			if (result == null)
				bw.write("{}");
			else {
				bw.write(serialize(result));
			}
			bw.flush();
		}
	}

	// CLIENT

	BufferedWriter bw;
	BufferedReader br;
	
	public void close() {
		try {
			bw.close();
		} catch (IOException e) {
		}
		try {
			br.close();
		} catch (IOException e) {
		}
	}

	@SuppressWarnings("unchecked")
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		if (bw == null || br == null) {
			Socket s = new Socket("localhost", port);
			bw = new BufferedWriter(new OutputStreamWriter(s.getOutputStream(), utf8));
			br = new BufferedReader(new InputStreamReader(s.getInputStream(), utf8));
			System.out.println("done");
		}
		bw.write(serMethod(method.getName(), args));
		bw.flush();

		HashMap<String, Object> parse = (HashMap<String, Object>) parse(br);

		Class<?> returnType = method.getReturnType();
		if (returnType.equals(Void.TYPE)) {
			return null;
		} else if (returnType.equals(String.class)) {
			return (String) parse.get(returnType.getSimpleName());
		} else if (returnType.equals(Double.class)) {
			return (Double) parse.get(returnType.getSimpleName());
		} else {
			return deSerialize(returnType.newInstance(), (HashMap<String, Object>) parse.get(returnType.getSimpleName()));
		}
	}

	private Map<String, Object> parseObject(Reader r) throws IOException {
		Map<String, Object> map = new HashMap<String, Object>();
		int i;
		while ((i = r.read()) >= 0) {
			char c = (char) i;
			if (Character.isWhitespace(c))
				continue;
			if (c == '}')
				break;
			if (c != '"')
				readUntil(r, '"');
			String name = readUntil(r, '"');
			readUntil(r, ':');
			if (name != null && name.length() > 0)
				map.put(name, parse(r));
			else
				Thread.dumpStack();
		}
		return map;
	}

	private List<Object> parseArray(Reader r) throws IOException {
		List<Object> list = new ArrayList<Object>();
		while (true) {
			Object v = parse(r);
			if (v == arrayEndIndicator)
				break;
			if (v == commaIndicator)
				continue;
			list.add(v);
		}
		return list;
	}

	private String readUntil(Reader r, char ch) throws IOException {
		StringBuffer sb = new StringBuffer();
		int i;
		while ((i = r.read()) >= 0) {
			char c = (char) i;
			if (c == ch)
				break;
			sb.append(c);
		}
		return sb.toString();
	}

	Object arrayEndIndicator = new Object();
	Object commaIndicator = new Object();

	public Object parse(Reader r) throws IOException {
		int i;
		while ((i = r.read()) >= 0) {
			char c = (char) i;
			if (c == '{')
				return parseObject(r);
			else if (c == '[')
				return parseArray(r);
			else if (c == ']')
				return arrayEndIndicator;
			else if (c == ',')
				return commaIndicator;
			else if (c == '"')
				return readUntil(r, '"');
			else if (Character.isDigit(c) || (c == '-'))
				throw new RuntimeException("number format is unsupported");
			else if (Character.isWhitespace(c))
				continue;
		}
		return null;
	}

	public static <T> T deSerialize(T o, HashMap<String, Object> fields) {
		Field[] declaredFields = o.getClass().getDeclaredFields();
		for (Field field : declaredFields) {
			String fName = field.getName();
			try {
				field.set(o, fields.get(fName));
			} catch (IllegalAccessException e) {
				throw new RuntimeException(e);
			}
		}
		return o;
	}

	public static String serialize(Object o) {
		String type = o.getClass().getSimpleName();
		StringBuffer sb = new StringBuffer("{ \"" + type + "\" : ");
		if (o instanceof String) {
			sb.append("\"" + o + "\"");
		} else {
			sb.append("{\n");
			sb.append(serFields(o));
			sb.append("\n}");
		}
		return sb.append("\n}\n").toString();
	}

	public static String serFields(Object o) {
		StringBuffer sb = new StringBuffer();
		Field[] declaredFields = o.getClass().getDeclaredFields();
		for (Field field : declaredFields) {
			field.setAccessible(true);
			String fName = field.getName();
			try {
				Class<?> type = field.getType();
				if (type.equals(Integer.TYPE) || type.equals(Double.class)) {
					sb.append("\n\"" + fName + "\" : " + field.get(o) + ",");
				} else if (type.equals(String.class)) {
					sb.append("\n\"" + fName + "\" : \"" + field.get(o) + "\",");
				} else {
					System.out.println("WARN: unsupported type: " + type);
					sb.append("\n\"" + fName + "\" : \"" + field.get(o) + "\",");
				}
			} catch (IllegalAccessException e) {
				throw new RuntimeException(e);
			}
		}
		// remove last comma
		if (sb.charAt(sb.length() - 1) == ',')
			sb.setLength(sb.length() - 1);
		return sb.toString();
	}

	public static String serMethod(String name, Object[] params) {
		StringBuffer sb = new StringBuffer("{ \"" + name + "\" : [ ");
		if (params != null)
			for (Object object : params) {
				if (object instanceof String)
					sb.append("\"" + object + "\"");
				else
					sb.append(" { " + serFields(object) + " } \n");
			}
		return sb.append("]\n}\n").toString();
	}
}

```
