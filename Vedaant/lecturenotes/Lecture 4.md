```table-of-contents
```
# Persistent Data

- Simply just permanent data for applications (storage of data from working memory) 
	- Can be updated, just not as frequently as transient/volatile data 
	- Stored in DBs/SSDs/HDs etc 
- Why is it needed?
	- To allow data to be used and reused (save and reload)

- Choice of persistent method is part of design 
	- What does the application do/what kind of data it is
	- It should come out of the requirements stage of the software development life cycle (SDLC)
	- Corporate licenses, storage limitation, rapid access to data

- Additional aspects to consider when picking method:
	- Programming Agility
		- How easy and fast is it to develop, with little overhead?
	- Extensibility
		- Can the data easily be extended, e.g. by adding new fields or attributes?
		- CSV for example:
			- Hard to add new fields, needs new column on every file
		- Graph databases such as Neo4j considered most extensible
	- Portability
		- Will other applications access the data? Will it run on other hardware?
		- **XML** is the most resilient.
		- **JSON** relies partly on the implementation.
		- **CSV** gives no resilience: line endings and character representations can differ across systems.
	- Maintainability (Main reason for SDLC)
		- What is the lifetime of your solution, and what resourcing do you have for maintenance?
		- Bespoke formats need a paid software engineer every time you need access 
	- Robustness 
		- Is the format well designed and structured?
		- No schema means there's no way to verify the data is correctly formatted
			- Schema is description of what valid data looks like: what fields exist and what type each one is
		- A lack of schema leads to interoperability problems/lack of record matching
		- XML and JSON are standards, so they provide interoperability and robustness, vs a Bespoke method
	- Size vs Completness
		- Lossy vs lossless: audio and images vs financial or scientific data
		- Compressed data also needs compute to read back, this may be expensive
	- Internationalisation
		- ASCII vs UTF-8
		- Who will use the data? Know your audience.

## Databases

- DBMSs (Database Management Systems) are commonly used to store large volumes of data.
	- Examples: PostgreSQL, Oracle, SQL Server, MySQL, MariaDB
		- SQLite: runs on your own device
		- Apache Cassandra: for huge volumes of data
		- MySQL: relational, supports atomic transactions
	- B-trees are the data structure databases use to store and index data in pages on disk. They are covered later in the course.
- Relational DBs
	- Linking tables through unique identifiers avoids problems of repeating data entry

## Bespoke Implementation 
- Demo: implementing a simple logging application that stores in CSV to save/load log errors to/from a text file
- We will do this in a new class Bespoke
	- Fields are id, timestamp, error message, level, class name and method name
	- Custom SimpleLog class whose objects represent one line of our logging app
		- It only contains a constructor, getters and setters and a toString function
```java
	public String toString(String delimiter)
	{
		return this.getId() + delimiter + this.getDateTime() + delimiter + this.getErrorMessage() + delimiter + this.getLevel() + delimiter + this.getClassName() + delimiter + this.getMethodName();
	}
```
- This turns the object back into one line of text, with the fields joined by the text file delimiter (,)
	- getId returns an int, but int + String makes java convert number to text
	- This is an overload, not override of the default java toString() function
		- This is as toString() takes no arguments, this one takes a string making it a seperate method

Now for the bespoke class: 
```java
public class Bespoke {
	
	/*
	 * Contains information about log entries
	 * key = id, value = log object
	 */
	public Map<Integer, SimpleLog> data;
	
	public static void main(String[] args)
	{
		Bespoke b = new Bespoke();
		// Sets the delimiter for text files
		String delimiter = ",";

		// Load data from that input txt file into b.data
		b.loadData("./resources/listoferrors_in.txt", delimiter);

		// entrySet() returms every key-value pair in the map
		// So for loop gets all entries in the map
		for(Map.Entry<Integer, SimpleLog> e : b.data.entrySet()) 
			// And then prints them using the toString to turn it into a csv line
			System.out.println(e.getValue().toString(delimiter)); 

		// Writes to the output text file, true means append onto it
		b.saveData("./resources/listoferrors_out.txt", delimiter, true);
	}
```
- The map "data" holds all log entries in memory, with the key being the id.
	- data.get(101) would return entry with id 101
- Main is static and has no object of it's own so it initialises bespoke class with b

Now to define the Bespoke class, starting with save:
```java
/*
	 * Save Data into a File
	 * @param filePath
	 * @param delimiter (comma, tab, semi-colon, ...)
	 * @param append 
	 */
	public void saveData(String filePath, String delimiter, Boolean append) {

		try(BufferedWriter bw = 
		new BufferedWriter(new FileWriter(filePath, append)))
		// new FileWriter opens the file for writing, when append is true it starts writing at the end of the file. 
		// This is then wrapped in a BufferedWriter so text is written to disk in chunks with the buffer rather than one tiny write at a time
		// This is defined by bw, and put in a try so IOExceptions like missing folders or no permissions simply close bw and flush buffer without error
		{
			// Loop over every entry in map
			for(Map.Entry<Integer, SimpleLog> e : data.entrySet()) {
				// For each SimpleLog object, get the value from the Map
				SimpleLog p = e.getValue();
				// Convert it into the csv line format
				String record = p.toString(delimiter);
				// Write record to loaded file, and then create newLine, using newLine writes the platform's line ending, for example \n for Linux
				bw.write(record);
				bw.newLine();
				//bw.flush(); Not needed because try/catch does this
			}
		}catch(IOException e)
		{
			e.printStackTrace(); //Prints error where it happened
		}
	}
```
- There are many ways to write to a file in java
	- FileWriter, FileOutputStream, BufferedWriter etc
	- BufferedWriter is very common, has a buffer
		- which is good when you have IO operations, as it may achieve better performance

Loading data:
```java
public void loadData(String filePath, String delimiter)
	{
		 try(BufferedReader br = new BufferedReader(new FileReader(filePath)))
		 // Same writing method as last time
		 {
			// Holds current line in text file
			 String record;
			 // Create fresh empty map
			 data = new HashMap<Integer, SimpleLog>();
			
			// Iterate through file until last line
			 while((record = br.readLine()) != null)
			 {
				 // Split into tokens at each comma. token[0] for example contains simpleLog ID
				 String[] tokens = record.split(delimiter);
				  //Make sure we get exactly 6 fields, otherwise skip line
				 if(tokens.length != 6 ) continue;
				 // Convert id to string
				 int id = Integer.parseInt(tokens[0]);
				 // Store data in SimpleLog
				 SimpleLog p = new SimpleLog(id, tokens[1], tokens[2], tokens[3], tokens[4], tokens[5]);
				 data.put(id, p);
			 }
		 }catch(IOException e)
		{
			e.printStackTrace();
		}
	}
```

## Serialization 

- Instead of writing own loops and parsing logic, Java libraries allow you to persist whole objects as binary data
- Not everything can be serialised for example, like file pointers

- To serialize an object means to convert its state to a byte stream so that the byte stream can be reverted back into a copy of the object 
	- Java object is serializable if its class or any of its superclasses implements either the java.io.Serializable interface or its subinterface, java.io.Externalizable
- Deserialization is the process of converting the serialized form of an object back into a copy of the object.
	- For example, the java.awt.Button class implements the Serializable interface
	- So you can serialize a java.awt.Button object and store that serialized state in a file. 
	- Later, you can read back the serialized state and deserialize into a java.awt.Button object.

### Example 

- Implementing Serializable to store a name and a password
```java
public class PDSerialization implements Serializable{

	// this is the version of the object. Written into file and checked, changing number will fail the reading.
	private static final long serialVersionUID = 1L; 
	
	public transient int id; //transient/don't save this field, id never written to file
	public String name;
	public String password;
	
	// Constructor  
	public PDSerialization(String name, int id, String password)
	{
		this.name = name;
		this.id = id;
		this.password = password;
	}

	public static void main(String[] args) 
	{
		// Create the object pds to save
		PDSerialization pds_sv = new PDSerialization("Bernardo", 321, "myPassword");
		// And save it to the file. .ser is naming convention for serialized
		pds_sv.saveData("./resources/listofpeople.ser");

		// Load and print file
		PDSerialization pds_ld = loadData("./resources/listofpeople.ser");
		System.out.println("id: " + pds_ld.id + " - " + pds_ld.name + " - " + pds_ld.password);
		System.out.println(pds_ld);
		// Id will be 0 and not 321 because we never save it
	}


```
- We need the serial number to protect us from reading data into a class whose structure has changed 

SaveData: 
```java
public void saveData(String filePath)
	{
		// Same use of try catch as above
		try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(filePath))) 
		{
			// Automatically serializes and stores entire object
			oos.writeObject(this);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
```
- new FileOutputStream(filePath) opens the file for writing raw bytes, no append flag
- new ObjectOutputStream(...) wraps it as oos and knows how to turn object into bytes

LoadData:
```java
public static PDSerialization loadData(String filePath)
	{
		PDSerialization data = null;
		try(ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filePath)))
		{	
			// Cast needed to force conversion into PDSerialization type
			data = (PDSerialization) ois.readObject();
		} catch (IOException | ClassNotFoundException e) {
			e.printStackTrace();
		}

		return data;
	}

```
- Static since before loading we don't have an object, this creates and returns this

Few additional points about serialization: 
- Don't use for passwords etc, since it can be viewed from the outside
- Use transient to avoid storing things in the object that you don't want
- Class must implement Serializable 
- ArrayLists are serializable by default. So are hashmaps (check documentation)

Bespoke and Serialization: 
- Not a great solution for enduring persistence, neither is robust enough
	- Bespoke can be cross-platform, but can cause issues
- Serialization issues:
	- May depend on programming language, can't read Java-serialised objects in Python
	- Loses object references, open files etc
	- Security issues

## XML

- Robust solution: structured, platform-independent, usable across languages
- .docx files are represented using ZIP + XML 

XML Structure: 
![[file-Pasted image 20261005180314-20261005180315017.jpg|425]]
- Consists of a header with three parts:
	- version, encoding (UTF-8), standalone (whether there is a DTD, not covered in course) 
- Then just a textual description of a tree
	- Every tag has an opening tag` <child> and a closing tag </child>`
	- One root node, can be called anything
	- Root has children, which have their own children
- Metadata can go inside the tags as attributes, as seen in the first child above
- XML is case-sensitive 
- There are also reserved characters that will cause parser errors:
	- `<subchild> 10 < x < 100 </subchild>` is not allowed
	- You can use these instead:
	- ![[file-Pasted image 20261005181152-20261005181152300.jpg|391]]

### Implementing in Java

Two main approaches: 
![[file-Pasted image 20261005182339-20261005182339871.jpg]]
- DOM is taught and tested in this course

XML DOM Format:
![[file-Pasted image 20261005184028-20261005184028793.jpg|400]]

DOM Steps to save data to file:
1. Create a `DocumentBuilder` using a `DocumentBuilderFactory`.
2. Create a `Document` from the `DocumentBuilder`.
3. Create and append elements.
4. Transform the XML to a `Result` (the output file) using a DOM Transformer 

Steps to load XML/Dom
- DocumentBuilderFactory → DocumentBuilder → Document → object data.

### Example
- Creating a demo that persists a list of Person objects
- A person is a class wih an id, firstname and lastname. It includes an override of toString to return a string with all elements

Imports to make it work
```java
import java.io.File;
import java.util.ArrayList;
import java.util.List;

import javax.xml.parsers.DocumentBuilder;          // the parser
import javax.xml.parsers.DocumentBuilderFactory;   // makes parsers
import javax.xml.transform.OutputKeys;             // output settings (encoding, indent)
import javax.xml.transform.Transformer;            // writes a tree out
import javax.xml.transform.TransformerFactory;     // makes transformers
import javax.xml.transform.dom.DOMSource;          // "the input is a DOM tree"
import javax.xml.transform.stream.StreamResult;    // "the output is a file/stream"

import org.w3c.dom.Document;   // the whole XML document
import org.w3c.dom.Element;    // a tag, e.g. <Person>
import org.w3c.dom.Node;       // anything in the tree (element, text, ...)
import org.w3c.dom.NodeList;   // a list of nodes
```
- org.w3c.dom: W3C standard is used for DOM tree classes, named after W3C since it exists in other languages as well. 
```java
public class PDXML {
	List<Person> people;

	public PDXML()
	{
		people = new ArrayList<Person>();
	}
```
- People is an ArrayList of people to save
	- No access modifier, package-private, this is what allows main to add to it

Next the main function:
```java
public static void main(String[] args) 
	{
	//Create previously defined PDXML object, add three people to the list
		PDXML xml = new PDXML();
		xml.people.add(new Person(10,"Lisa", "Simpson"));
		xml.people.add(new Person(11,"Homer", "Simpson"));
		xml.people.add(new Person(12,"Maggie", "Simpson"));
		
		// Write those three to the output xml file
		xml.saveData("./resources/people_out.xml");
		
	// Reads a different file and prints them out for each person in list
		List<Person> newPeople = xml.loadData("./resources/people_in.xml");
		for(Person p : newPeople)
		{
			System.out.println(p.toString());
		}
	}
	
/* This outputs the xml file and this in systemout:
id: 1 - Firstname: Johnny - Lastname: B. Good
id: 2 - Firstname: Bart - Lastname: Simpson
*/
```

Saving data: 
```java
public void saveData(String filePath)
	{
		// File object is just a path, doesn't open or create anything yet
		File f = new File(filePath);
		
		//Factory Pattern: Can't define DocumentBuilder from library directly since it is an abstract class, dbf picks right implementation for this java installation from library
		DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

		try {
			//DocumentBuilder is the XML parser that parses files into documents or makes empty documents
			DocumentBuilder db = dbf.newDocumentBuilder(); 
			// Creates empty document in memory for future use
			Document d = db.newDocument();

			//create root element of XML tree and append to empty doc
			Element rootElement = d.createElement("People");//<People>
			d.appendChild(rootElement); 

	//loop through all people to create element for each person in list
			for(Person person : people)
			{
				// Creates: //<Person id="1">..
				Element personElement = d.createElement("Person"); 
				personElement.setAttribute("id", Integer.toString(person.getId()));
				
			/get the list of nodes by tag name (all <person> items)/<FirstName> ... </FirstName>
				Element firstnameElement = d.createElement("FirstName"); 
			// add <FirstName> here goes firstname </FirstName>
	firstnameElement.appendChild(d.createTextNode(person.getFirstname()));
	// add it to Person element: 
	//<Person id="1"><FirstName> here goes firstname </FirstName></Person> 
				personElement.appendChild(firstnameElement);
			// Do the same for lastname
				Element lastnameElement = d.createElement("LastName");
		lastnameElement.appendChild(d.createTextNode(person.getLastname()));
				personElement.appendChild(lastnameElement);
				// Finally append  person as child of root
				rootElement.appendChild(personElement);
			}

			// Same factory method for the transformer
			// Transformer converts the created tree into XML file
			Transformer transformer = TransformerFactory.newInstance().newTransformer();
			// Use UTF-8
			transformer.setOutputProperty(OutputKeys.ENCODING, "utf-8"); 
			//Add line breaks and indentation to make it human readable
			transformer.setOutputProperty(OutputKeys.INDENT, "yes");

			//DOMSource and StreamResult are wrappers 
			// DOMSource validates our tree document d as a DOM Tree
			DOMSource source = new DOMSource(d); 
			// Output validates as text file
			StreamResult result = new StreamResult(f);
			// Transform actually does the work
			transformer.transform(source, result); 
			
		}
		catch(Exception e)
		{
			e.printStackTrace();
		}
	}
```
- Most of these methods are derived from the imported libraries, read their documentation for additional info
- .createTextNode also escapes reserved characters automatically 
The resulting XML from this code:
```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<People>
    <Person id="10">
        <FirstName>Lisa</FirstName>
        <LastName>Simpson</LastName>
    </Person>
    <Person id="11">
    ...
</People>
```
- Reminder: node is an element, like FirstName and is part of the tree. Has opening and closing tags, and can contain text or child elements 
- An attribute like id=1, is a name-value pair written inside an element's opening tag. Just meta, cannot have children of it's own

Now to load data from a file:
```java
public List<Person> loadData(String filePath)
	{
		// Same as before
		File f = new File(filePath); 
		DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
		//New list to hold loaded people
		List<Person> lp = new ArrayList<Person>();
		
		try {
			DocumentBuilder db = dbf.newDocumentBuilder();
			// Read entire file
			Document d = db.parse(f); 
			// Remove spaces, line breaks etc
			d.getDocumentElement().normalize();
			//Create nodelist, and get every Person element in document order at any depth
			NodeList nl = d.getElementsByTagName("Person");
			// NodeList is not a java list, so needs custom loop
			// For each person element do the following:
			for(int i = 0; i < nl.getLength(); i++)
			{
				// Get node item from list
				Node n = nl.item(i); 
				// Ensure you only get node and not other elements
				if(n.getNodeType() == Node.ELEMENT_NODE) {
				// Node is generic, we need to convert it to Element to use the getAttriubte and other functions
					// We can then use the methods from lib to get data
					Element element	= (Element) n;
					Integer id = Integer.parseInt(element.getAttribute("id"));
					String firstname = element.getElementsByTagName("FirstName").item(0).getTextContent();
					String lastname = element.getElementsByTagName("LastName").item(0).getTextContent();
					// Form as Person type and add to return list
					Person p = new Person(id, firstname, lastname);
					lp.add(p);
				}
			}
		}
		catch(Exception e)
		{
			e.printStackTrace();
		}
		
		return lp;
	}
 
```

### Conclusion 

XML Pros and Cons

Pros:
- Robust and extensible
- Human-readable
- Portable: platform-independent and programming-language-independent
- Supports Unicode (international encoding)
- Easy format verification
- Can represent data structures (trees, lists, …)
Cons:
- Syntax is verbose and redundant. It is the most verbose option in this lecture.
- File sizes are usually big because of this.
- Does not explicitly support arrays or `null`.
    - Lists are represented as many children.
    - There is no concept of null.
- Can only represent lists and trees, though metadata can be stored as attributes.

## JSON

- **JSON** = JavaScript Object Notation.
- Open standard format, widely used
- String representation of objects, built around attribute–value pairs
	- Produces smaller, more readable documents than XML
```json
[{"age":11,"name":"Bart"},{"age":40,"name":"Homer"}]

{"attribute-name": {JSON object}}   // nested object
{"attribute-name": "string"}
{"attribute-name": [array]}
{"attribute-name": 1}               // number
{"attribute-name": true}            // boolean
{"attribute-name": null}
```
- It can have nested objects, and a variety of types

- Json is not natively supported in Java, we are using Gson (google's JSON library)
- Despite this, we don't need any tree-walking boilerplate like XML

We can begin by defining the classes of the data we want to store:
```java
import com.google.gson.annotations.SerializedName;

public class AddressJSON {
	//@SerializedName("Cidade")
	private String city;
	private String country;
	
	// Constructor 
	public AddressJSON(String city, String country)
	{
		this.city = city;
		this.country = country;
	}
	
	// Normal getters and setters
	public String getCountry() {
		return country;
	}

	public void setCountry(String country) {
		this.country = country;
	}

	public String getCity() {
		return city;
	}

	public void setCity(String city) {
		this.city = city;
	}
	
	// toString override like before
	@Override
	public String toString(){
		return city + ", " + country;
	}
}
```
- SerializedName is commented but if it wasn't, java would use city as the key, but json would use Cidade
	- By default Gson would use the field name as the json key
	- This can be used for localization etc 
	- When loading, cidade values still fill city key and vice versa

Same with PersonJSON:
```java

public class PersonJSON {
	private int id;
	private String firstname;
	private String lastname;
	private AddressJSON address; // Nested JSON Object
	
	// Constructor
	public PersonJSON(int id, String firstname, String lastname, AddressJSON address)
	{
		this.id = id;
		this.firstname = firstname;
		this.lastname = lastname;
		this.address = address;
	}

	public int getId() {
		return id;
	}

	// Other getters go here, nothing special
	
	// Same string function
	public String toString()
	{
		return "id: " + this.id + " - Firstname: " + this.firstname + " - Lastname: " + this.lastname + ", " + getAddress().toString();
	}
}
```

We can now implement these classes to store JSON persistently: 
```java
import java.io.FileReader;
import java.io.FileWriter;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.List;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.google.gson.reflect.TypeToken;
import com.google.gson.stream.JsonReader;

public class PDJSON {
	
	// Create list of people with JSON data class
	private List<PersonJSON> people;
	
	// Constructor 
	public PDJSON() {
		people = new ArrayList<PersonJSON>();
	}
	
	public static void main(String[] args) {
		PDJSON pdj = new PDJSON();
		
		// Add three people and adresses to list
		pdj.people.add(new PersonJSON(1,"Bart", "Simpson", new AddressJSON("Springfield", "USA")));
		pdj.people.add(new PersonJSON(2,"Homer", "Simpson", new AddressJSON("Springfield", "USA")));
		pdj.people.add(new PersonJSON(3,"Mickey", "Mouse", new AddressJSON("Orlando", "USA")));
		
		// Save list to file
		pdj.saveData("./resources/listofpeople.json");

		// Load same file back in temporary file
		List<PersonJSON> lp = pdj.loadData("./resources/listofpeople.json");
		// Print each one
		for(PersonJSON pj : lp)
		{
			System.out.println(pj.toString());
		}
	}
	
/*
Output would be:
id: 1 - Firstname: Bart - Lastname: Simpson, Springfield, USA
id: 2 - Firstname: Homer - Lastname: Simpson, Springfield, USA
id: 3 - Firstname: Mickey - Lastname: Mouse, Orlando, USA 
*/
```

Now to actually save the data:
```java
	public void saveData(String filePath) {
		//Gson gson = new Gson(); would write everything to one line
		//GsonBuilder with these options adds line breaks and indentation
		Gson gson = new GsonBuilder().setPrettyPrinting().create();
		
// Open the file, don't need BufferedWriter because Gson buffers internally 
		try(FileWriter fw = new FileWriter(filePath)){
			// Single command converts entire list into json array
			gson.toJson(people, fw);		
		}catch(Exception e)
		{
			e.printStackTrace();
		}
	}
```
- No need for advanced nesting logic like XML, just one command which:
	- The `List` becomes a JSON array `[ ... ]`
	- Each `PersonJSON` becomes an object `{ ... }`, with keys from the field names
	- Each nested `AddressJSON` becomes a nested object, automatically
	- `int` is written as a JSON number and `String` as a JSON string
This creates the following json:
```json
[
  {
    "id": 1,
    "firstname": "Bart",
    "lastname": "Simpson",
    "address": {
      "city": "Springfield",
      "country": "USA"
    }
  },
  ...
]
```

Now to load the data:
```java
	public List<PersonJSON> loadData(String filePath) {
		// Plain gson is enough since we don't need human readable input
		Gson gson = new Gson();
		// JsonReader is gson class for reading Json. Intilized as null here to allow access for the return type. Not initilized fully so that we can do that in try catch for errors
		JsonReader jsonReader = null;
		
		// Explained below
		final Type CUS_LIST_TYPE = new TypeToken<List<PersonJSON>>(){}.getType(); 
		
		try{
			// jsonReader defined and pointed to filepath
			jsonReader = new JsonReader(new FileReader(filePath));
		}catch (Exception e) {
			e.printStackTrace();
		}
		//gson reads file with object type defined and returns as List
		return gson.fromJson(jsonReader, CUS_LIST_TYPE);
	}
```
Final Type explained:
- To load the file, Gson has to know what type to build
	- JSON file gives array of objects, doesn't tell us they are PersonJSONs 
	- We can't simply just say `List<PersonJSON>` either as generics only exist at compile time, at run time it only exists as a list (known as type erasure)
- final Type CUS_LIST_TYPE
	- This is defining an object whose only job is to describe a type. Final and capitalised to follow convention for a constant
- `new TypeToken<List<PersonJSON>>() {}` bypasses this
	- The `{}` at the end creates an **anonymous subclass** of `TypeToken`: a tiny unnamed class that extends `TypeToken<List<PersonJSON>>`.
	- This is the same as writing
	- ```java
	  class PersonListToken extends TypeToken<List<PersonJSON>> { }   // a tiny class
	  Type CUS_LIST_TYPE = new PersonListToken().getType();          // make one, ask for its type
	  ```
	  - This works because Java forgets the < > on an ordinary object but remembers what a class extends, including the < >.
	  - So `CUS_LIST_TYPE` now holds the full description **"List of PersonJSON"**

### Conclusion 

Json Pros and Cons

Pros: 
- More lightweight.
- Straightforward to implement.
- **Supports arrays and null.**
- Can easily distinguish boolean, number and string types.
- Data is available as JSON objects.

Cons:
- Lacks some language features of XML, e.g. XML attributes.
- No native support in Java. XML is 100% compatible.
- No display capabilities: it is not a markup language.