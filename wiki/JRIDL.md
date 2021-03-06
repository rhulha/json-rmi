#labels JRIDL,RPC,JSON,RMI,Java

```Java

package org.json.rmi;

import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.Reader;
import java.nio.charset.Charset;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
import java.util.Set;

public class JRIDL {

	private final String servicename;
	private final String filename;

	public JRIDL(String servicename, String filename) {
		this.servicename = servicename;
		this.filename = filename;
	}

	public void gen(String path) throws IOException {

		Reader fr = new BufferedReader(new InputStreamReader(new FileInputStream(filename), Charset.forName("utf-8")));
		JSON_RMI p = new JSON_RMI();
		Object parse = p.parse(fr);
		fr.close();

		@SuppressWarnings("unchecked")
		HashMap<String, Object> desc = (HashMap<String, Object>) parse;

		@SuppressWarnings("unchecked")
		HashMap<String, Object> objects = (HashMap<String, Object>) desc.get("objects");

		@SuppressWarnings("unchecked")
		HashMap<String, Object> methods = (HashMap<String, Object>) desc.get("methods");

		Set<Entry<String, Object>> entrySet = objects.entrySet();
		for (Entry<String, Object> entry : entrySet) {
			genObject(path, entry);
		}

		StringBuffer sb = new StringBuffer();
		sb.append("package gen;\n");
		sb.append("import java.io.IOException;\n");
		sb.append("public interface " + servicename + " {\n");

		entrySet = methods.entrySet();
		for (Entry<String, Object> entry : entrySet) {
			@SuppressWarnings("unchecked")
			HashMap<String, Object> params = (HashMap<String, Object>) entry.getValue();
			sb.append("public " + convertType(params.get("returns").toString()) + " " + entry.getKey() + " ( ");

			@SuppressWarnings("unchecked")
			List<Map<String, Object>> paramsList = (List<Map<String, Object>>) params.get("parameter");
			for (Map<String, Object> paramEntry : paramsList) {
				//@SuppressWarnings("unchecked")
				//HashMap<String, Object> params = (HashMap<String, Object>) object;
				Entry<String, Object> next = paramEntry.entrySet().iterator().next();
				sb.append(convertType(next.getValue().toString()) + " " + next.getKey() + ",");
			}
			// remove last comma
			if (sb.charAt(sb.length() - 1) == ',')
				sb.setLength(sb.length() - 1);
			sb.append(") throws IOException;\n");
		}
		sb.append("}\n");

		FileOutputStream fos = new FileOutputStream(path + servicename + ".java");
		fos.write(sb.toString().getBytes(Charset.forName("utf-8")));
		fos.close();
	}

	String convertType(String fieldType) {
		return fieldType;
	}

	private void genObject(String path, Entry<String, Object> obj) throws IOException {

		StringBuffer sb = new StringBuffer();
		sb.append("package gen;\n");
		sb.append("public class " + obj.getKey() + " {\n");

		@SuppressWarnings("unchecked")
		HashMap<String, Object> fields = (HashMap<String, Object>) obj.getValue();

		Set<Entry<String, Object>> entrySet = fields.entrySet();
		for (Entry<String, Object> entry : entrySet) {
			String fieldType = convertType(entry.getValue().toString());
			sb.append("public " + fieldType + " " + entry.getKey() + ";\n");
		}

		sb.append("}\n");

		FileOutputStream fos = new FileOutputStream(path + obj.getKey() + ".java");
		fos.write(sb.toString().getBytes(Charset.forName("utf-8")));
		fos.close();
	}

}

```
