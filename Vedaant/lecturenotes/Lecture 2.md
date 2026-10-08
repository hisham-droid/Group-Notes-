```table-of-contents
```
# UML 

- A language for modelling systems in graph mode
- Designed to be "natural" / easy to understand and use.
- Used for **visualising, specifying, constructing, and documenting** software artifacts.
    - Lets you understand a system from **various perspectives**.
    - Used for documentation of requirements, design, architecture, etc.
- Timing of use:
    - Sometimes used **before** coding: _forward design_.
    - Sometimes used **after** coding: _documentation / backward design_.
- Made with building blocks:
	- Things: elements being modelled (classes)
	- Relationships: ties the things together 
	

## Class Diagram
- Shows a set of classes, interfaces and collabrations and their relationships 
	- OOP abstraction of real-world entities 
![[file-Pasted image 20260817224215-20260817224215044.jpg|700]]
- We can add types and intial values in UML. The + and - represent the visibility of the attribute. UML can be made as complicated as we want

## Visibility 
![[file-Pasted image 20260817224720-20260817224720321.jpg|450]]
- Default is package private 
- Protected: field/method invisible to random classes, but visible to your class and any subclass (including subclasses in other packages)
- Package: visible to any other class in the same package 

Example:
![[file-Pasted image 20260817225233-20260817225233867.jpg|450]]
- Student inherits from person and then adds their own attributes and methods

## Relationships 

![[file-Pasted image 20260817225324-20260817225324093.jpg|500]]
- Enrols in is a relationship between Student and Course 

Types of relationships: 

| Relationship                     | Meaning                                                                                                    | Informal phrase       | Notation                             |
| :------------------------------- | :--------------------------------------------------------------------------------------------------------- | :-------------------- | :----------------------------------- |
| **Generalisation (Inheritance)** | More-general class → more-specialised class. Child inherits all attributes, operations, relationships      | _is-a / is-like-a_    | Solid line, **hollow triangle** head |
| **Association**                  | Structural relationship between instances; each end has properties (e.g., multiplicity)                    | _works-for_           | Solid line                           |
| **Dependency**                   | Runtime relationship; one object/class uses another (often a method parameter)                             | _uses-a / depends-on_ | **Dashed** line, open arrow          |
| **Aggregation**                  | _has-a_ / _is-part-of_; collection of objects where object can exist outside the collection                | _has-a_               | **Hollow diamond**                   |
| **Composition**                  | Whole-part; collection of objects where object can not exist outside the collection (stronger aggregation) | _is-entirely-made-of_ | **Filled diamond**                   |

Inheritence: 
![[file-Pasted image 20260817230022-20260817230022045.jpg]]
- Different types of arrows represent what type of class is being inherited. 

Dependency: 
![[file-Pasted image 20260817230553-20260817230553714.jpg]]
- This is a temporary or weak relationship
- One class uses another class to perform a function, usually as method parameter, local variable or return value
- Printer uses Report as parameter, CarFactor depends on car in the same way 

Association:
![[file-Pasted image 20260817231004-20260817231004909.jpg]]

Composition: 
![[file-Pasted image 20260817231049-20260817231049271.jpg]]
- Item objects are created and shared by Order class, cannot exist without it

Aggregation:
![[file-Pasted image 20260817231138-20260817231138884.jpg]]

### Multiplicity 
![[file-Pasted image 20260817231212-20260817231212034.jpg]]
- Can be attached to any relationship 

### Java Inheritence 

- Subclass inherits features (attributes and operations from Superclass)
	- Features can therefore be reused instead of being written again and again
	- Subclass can extend and add new attributes and methods
	- Parent-Child relationship 
	- Can have multi-level inherietence:
	- ![[file-Pasted image 20260817231800-20260817231800777.jpg|225]]
- We can use the instanceof function to check what an object is an instance of:
	- `f instanceof Ferrari` → **true**
    - `f instanceof Car` → **true**
    - `f instanceof Vehicle` → **true**
    - `f instanceof Object` → **true** (Object is the superclass of every class)
    - `f instanceof Motorcycle` → **false** (parallel branch)

- Inheritence also allows a subclass to override a superclass method (unless it is of type final)
- Redefines superclass behavior to meet subclass requirements 

## Polymorphism 

- Different behaviours in different situations 
	- Java solves this by allowing **generics / type parameters** 
![[file-Pasted image 20260817232401-20260817232401148.jpg]]

- UML cannot show polymorphism, only inheritence
- Difference between two:
	- Inhertience (Static)
		- Reusable, inherients parent class features
		- New classes created on parent type
	- Polymorphism (Dynamic: Decided at runtime)
		- Object decides what form to take 
		- Applied to operations 

# Design Patterns 

- Design patterns are repeatable solutions to recurring problems in software design
	- Reuse solutions that have worked in the past
	- Capture design experience in a form people can use effectively
- Design patterns do not give exact answers, but are still more concrete than abstract principles

- Designing is the most challenging activity in software development
- No algrotihm from derving abstract solution models from requirements
- Design pattern is problem-solution pair: mapping from spefic design problem to generic solution

Design patterns are classified by purpose and scope:
- **Purpose:**
    - **I. Creational** — object creation or simply How do objects get made?
    - **II. Structural** — composition of classes/object or simply How are objects put together?
    - **III. Behavioural** — algorithms and responsibility assignment or simply How do objects talk to each other, and who is responsible for what?
- **Scope:**
    - **a. Class** — relationships via **inheritance** (fixed at compile time).
    - **b. Object** — relationships via **object composition** (can change at runtime).
- There are hundreds of patterns; GoF captured a foundational subset.
![[file-Pasted image 20260818000000-20260818000000704.jpg|375]]

## Creational Patterns 
- Instantiation process is abstracted 
	- Instead of making a new class in twenty places, we use a helper
- Help make a system indepdent of how its objects are created, composed and represented 
	- Class creational patterns use inheritance to vary the class instantiated
	- Object creational patterns delegate instationation to another object

### Factory Method

Intent:
- Define a method to create objects of the right kind
- Method does not hard code what kind, leaves that logic to a subclass
Use when:
 - A class can't know in advance which class of objects it must create
 - You want to keep the "which subclass?" decision in **one place**

![[file-Pasted image 20260818215028-20260818215028468.jpg]]

Assume you want to send notifications of different kinds:
```java
// scattered all over your codebase:
Notification n; 
if (channel.equals("SMS")) n = new SMSNotification(); 
else if (channel.equals("EMAIL")) n = new EmailNotification(); 
else n = new PushNotification(); 
n.notifyUser();
// ...same block copy-pasted in 20 places
```
- If we make a new type we have to edit this block in 20 places. 

This is why we use the factory method:
```java
public interface Notification {
    void notifyUser();   // no body — just a promise that this method exists
}

public class SMSNotification implements Notification {
    @Override
    public void notifyUser() {
        System.out.println("Sending an SMS Notification!");
    }
}
public class EmailNotification implements Notification {
    @Override
    public void notifyUser() {
        System.out.println("Sending an E-Mail Notification!");
    }
}
// PushNotification is the same shape
```
- This defines an interface and a concrete class for each type of notification 

- We can then move the if chain/logic about which notification class to create into one method, the NotificationFactory: 
```java
public class NotificationFactory {
    public Notification createNotification(String channel) {
    
        if ("SMS".equalsIgnoreCase(channel))   
        return new SMSNotification();
        
        if ("EMAIL".equalsIgnoreCase(channel)) 
        return new EmailNotification();
        
        if ("PUSH".equalsIgnoreCase(channel))  
        return new PushNotification();
        
        return null;
    }
}
```
- The data type is for createNotification is the notification interface
	- This allows any of the email, sms or pushnotification classes to be returned
	- This is why we need the notification interface over the subclasses 

Now we can update the callers to use the new code:
```java
NotificationFactory factory = new NotificationFactory(); // Only needs to be defined once

Notification n = factory.createNotification("EMAIL");
n.notifyUser();   // prints "Sending an E-Mail Notification!"
```
- This means a new type/editing specifics of the type can happen at one place only in the notificationFactory
- Callers only know the interface, they don't know details about specific classes
- This is not useful if we are manually writing the channel type, but if that is determined via code, for example something like:
	- `Notification n = factory.createNotification(user.getNotiChannel);`
	- This becomes extremely useful when we add new types

### Singleton 

Brief revision on static fields:
- A class is a blueprint. Objects are the things built from it.
	- Normal fields: every object gets it's own copy
	- Static field: there is only one copy attached to the class, shared by all objects
- Static fields exist as soon as the class is loaded, without any objects being created

Intent:
- Ensure that a class only has one instance and provide a global access point to it
Use When:
- There must be exactly one instance, and we don't want people to make new ones
- Examples: file system, print queue, database connections, configuration/security manager.

```java
public class SingletonConnection {

    // 1. the single shared instance for all objects, held in a static field
    private static SingletonConnection instance = null;

    // 2. private constructor → nobody OUTSIDE this class can call `new`
    private SingletonConnection() {}

    // 3. the only public way to get the instance. Also static since it is called on the class and not a single object
    public static SingletonConnection getInstance() {
        if (instance == null) {                      
        // if instance is null, it has not been created, so build it inside the class only the first time
            instance = new SingletonConnection();    
        }
        return instance;                            
         // otherwise every call returns the SAME object
    }
}
```
- The constructor being private means nobody outside class can call it and therefore create new one
- getInstance() is the only way in. Creates the instance lazily on first call, returns the same one afterwards.
```java
SingletonConnection db = new SingletonConnection();
// Causes a compile error since constructor is private 
SingletonConnection a = SingletonConnection.getInstance();  
// builds it
SingletonConnection b = SingletonConnection.getInstance();  
// returns the same one
// a and b point to the exact same object in memory
```

We can also have static initalizer blocks in classes (these will be useful later):
```java
static {
    // runs ONCE, automatically, when the class is first loaded
}
```
- This only runs once when class is first loaded, before main or any new
	- Order: static fields → static block (class load) → `main` → instance fields → constructor (each `new`)
- It can only touch static members

We can rewrite Singleton with this to not be lazy:
```java
private static SingletonConnection instance;
static { instance = new SingletonConnection(); }    // created at class load, no null check needed
public static SingletonConnection getInstance() { return instance; }
```

## Behavioural Patterns 

- Patterns concerned with algorithms and the assignment of responsibilities between objects
- Who does which job, who tells whom when something happens, and how a task is split up
- Two different scopes:
	- Class-Scope
		- Use inheritance to split up behaviour 
		- **Template Method**: the parent class owns the algorithm's steps, and subclasses fill in some of them
	- Object-Scope
		- Uses composition: separate objects cooperate on a task none of them could do alone
		- Observer: one object announces changes, and others react

### Observer 

Intent 
- Define a one-to-many dependency between objects
	- One thing (Subject) has many things (Observers) that care about it
- So when one object (the subject) changes state, all dependants (observers) are notified and updated automatically 
Use when: 
- You have some data and something that reacts to that data, and you want them as separate objects
- Change to one object requires changing others
- An object should be able to notify other objects without making assumptions about who they are
![[file-Pasted image 20260825124604-20260825124604755.jpg|425]]

The Problem: 
- We have a COVID check-in app. When a place gets a confirmed case, everyone who checked in there must be altered
- Naive way is to have the place call each person directly 
```java
public class Place {
    public void setCorona() {
        dylan.alert("McDonalds has a case");
        john.alert("McDonalds has a case");
        // new customer checks in? → edit this code and recompile!
    }
}
```
- Place must know about specific people, and the customer class. This is tight coupling that we want to avoid 
- You also can't add or remove people when the program is running

This can be solved by using an Observer: 

Creating interfaces for Observer and Subject:
```java
public interface Subject {
    void attach(Observer o);        // subscribe observer to subject
    void detach(Observer o);        // unsubscribe observer to subject
    void notifyAllObservers();      // broadcast to everyone subscribed
}

public interface Observer {
    void update(String msg);        // "something happened, here's the news"
}
```

Defining the subject (it holds a list of whoever is observing):
```java
public class Place implements Subject {
    private ArrayList<Observer> observers = new ArrayList<>();
    private String name;
	
	// Constructor 
    public Place(String name) { this.name = name; }

    public void attach(Observer o) { 
    observers.add(o); // Add to list of observers 
    }
    public void detach(Observer o) { 
    observers.remove(o); // Remove from list of observers 
    }

    public void notifyAllObservers() {
	    // For every person in the list of observers
        for (Observer obs : observers) {          
            obs.update(name + " has a confirmed case."); //tell each of them
        }
    }
}
```
- Place has no idea who it is notifing, we can add any observer without affecting place

Defining the Observer, decides what to do when told: 
```java
public class Customer implements Observer {
    private String name;
    
    // Constructor 
    public Customer(String name) { 
    this.name = name; 
    }
    
    public void update(String msg) {
        System.out.println("Hey " + name + "! Message for you: " + msg);
    }
}
```

Assuming the system detacts a case of corona and notifies all customers at place:
```java
public class ObserverDemo {
    public static void main(String[] args) {
        Customer c1 = new Customer("Dylan");
        Customer c2 = new Customer("John");
        Place mcd = new Place("McDonalds");
        mcd.attach(c1);  
        mcd.attach(c2);  // McDonalds: [Dylan, John]
        mcd.notifyAllObservers();  // notifies Dylan, John
        mcd.detach(c1);                     // McDonalds: [John]
        System.out.println("NEW CASE!");
        mcd.notifyAllObservers();  // notifies John only

    }
}
```
- Observers can unsubscribe/reattach at any time 
	- `Place` only knows the `Observer` _interface_, not `Customer`, so it's loosely coupled, and multiple different classes can be used as Observer
	- One observer can watch many subjects

### State 

Intent:
- Let an object change its behaviour based on its internal state, as if it changed class.
	- Same object responds to the same method call differently depending on what "mode" it's in
Use When: 
- Object has identifiable states and behaves differently in each
- Best for finite state machine (Programs that have fixed set of modes, plus rules for moving between them)
	- Examples: Logged in -> Logged out, draft -> published, playing -> paused etc

![[file-Pasted image 20260825134253-20260825134253459.jpg|325]]

The Problem: 
- Naive way is a stored boolean with an if in every method: 
```java 
public class UserSession {
    boolean loggedIn = false;

    public UserActivity createPost(String content) {
        if (!loggedIn) { 
        System.out.println("You cannot create post..."); 
        return null; 
        } else {
        // Create the post
        }
    }
    public boolean likePost(Integer id) {
        if (!loggedIn) { 
        System.out.println("You cannot like post..."); 
        return false; 
        } else {
        // Actually like the post
        }
    }
	// This then has to be done for every single method where the state is important 
}
```
- The same if check has to be copy-pasted into every method
- Added a new state will mean editing all of these

- State fixes this by making one class per state
- This holds all the behaviour for that state
- Main object just holds which state I am in, and passes every call to it

Structure:
```java
public abstract class UserState {                 
	// Field that will be used by all implementing classes
	// every state remembers which session it belongs to
    protected UserSession userSession; 
	
	// Constructor that will be used by all implementing classes
    public UserState(UserSession userSession) {    
        this.userSession = userSession;
    }

    public abstract boolean login(String username, String password);   
    public abstract UserActivity createPost(String content);           
    public abstract boolean logout();
}
```
- Abstract class means that you cannot create one directly, but you can still use it as a type 
	- Interface allows the same thing, but doesn't let us define the constructor or field which we need for every implementing class
	- This makes abstract class more efficient as those don't need to be rewritten for every single state
	- All implementing class must define what happens for both functions in that state

Now we must implement these for each state, to define how the app behaves when in each state.

Starting with logged out:
```java
class LoggedOutState extends UserState {
	// Stores the session that created it
    public LoggedOutState(UserSession userSession) {
        super(userSession);
    }

    @Override
    public boolean login(String username, String password) {
	    // Placeholder login system to login
        if ("admin".equals(username) && "123".equals(password)) {
            System.out.println("Login done");
            // Change userSession state to loggedIn by creating and passing             new created LoggedInState class
            userSession.changeState(new LoggedInState(userSession));
            return true;
        }
        // Wrong password:
        System.out.println("Login failed");
        return false;
    }
	
	// Nothing happens when we try to createPost or log out when logged out
    @Override
    public void createPost(String content) {
        System.out.println("You cannot create a post when logged out");
    }

    @Override
    public boolean logout() {
        System.out.println("You cannot log out when logged out");
        return false;
    }
}
```

Logged in State:
```java
class LoggedInState extends UserState {
    public LoggedInState(UserSession userSession) {
        super(userSession);
    }
	
	// Nothing happens when you try to log in when logged in
    @Override
    public boolean login(String username, String password) {
        System.out.println("You are already logged in");
        return false;
    }
	
	// Successfully create post
    @Override
    public void createPost(String content) {
        System.out.println(userSession.getUsername() + " posted: " + content);
    }

    @Override
    public boolean logout() {
        System.out.println("Logout done");
        // Change userSession state to new LoggedOutState
        userSession.changeState(new LoggedOutState(userSession));
        return true;
    }
}
```

Now we can implement the object that we actually use. Holds the current state and passes every action to it:
```java
class UserSession {
    private UserState userState; // Holds curren state, either a LoggedOutState or a LoggedInState object
    private String username; // Remembers who is logged in
	
	// Constructor, create UserSession object logged out
    public UserSession() {
        changeState(new LoggedOutState(this));
    }
	
	// Update state using this fuction, by simply replacing old with newState passed in as input
    public void changeState(UserState newState) {
        userState = newState;
    }
	
    public String getUsername() {
        return username;
    }
	
	// Pass action to state, and update username if approved
    public boolean login(String username, String password) {
        boolean ok = userState.login(username, password);
        if (ok) {
            this.username = username;
        }
        return ok;
    }
	
	// Pass action to state
    public void createPost(String content) {
        userState.createPost(content);
    }
	
	// Pass action to state, and remove username if approved
    public boolean logout() {
        boolean ok = userState.logout();
        if (ok) {
            this.username = null;
        }
        return ok;
    }
```

This can then be run to get the following output:
```java
UserSession s = new UserSession();
s.createPost("hello"); // `You cannot create a post when logged out`
s.login("admin", "999"); // Login failed
s.login("admin", "123"); // Login done
s.createPost("hello"); // admin posted: hello
s.login("admin", "123"); // You are already logged in
s.logout(); // Logout done
s.createPost("hello again"); // `You cannot create a post when logged out`
```
### Template Method 
Intent:
- Define the skeleton of an algorithm, deferring some steps to subclasses without letting them change overall structure 
Use when:
- Clients need only extend particular steps of algorithm, but not not the whole algorithm
	- Overall process is always the same. Only one or two steps differ between versions
- Report Generation uses this, template method fixes sequence, only parse step varies 
- Helps reduce repeated code 

![[file-Pasted image 20260825141114-20260825141114809.jpg|275]]

The parent is an abstract class.template method with the shared steps filled in: 
```java
public abstract class ActionReport {

    // STEP 1 (concrete, shared): read file into a String
    public String readData(String filePath) throws IOException {
        // Get the path as an object
        Path path = Paths.get(filePath);     
        // read file as array of bytes     
        byte[] bytes = Files.readAllBytes(path);  
        // Convert bytes that contain file content into strings and return
        String fileContent = new String(bytes);   
        return fileContent;
    }
    // The throws IOException means this might fail, and passes burden on handling that to the caller

    // STEP 2 (ABSTRACT, blank step that implementers must fill): turn the string into UserActivity objects (one per line: username, action, content, id).
    // Blank as format varies for CSV and XML, parent doesn't care
    public abstract List<UserActivity> parseFileContent(String rawData);

    // hook (given by default, shared but can be overriden to handle errors differently)
    public void handleException(Exception e) {
        System.out.println("Oops. Something went wrong");
    }

    // STEP 3 (concrete, shared): count how many times each action appears
    // Outputs list of type ActionCount, inputs userActivityList created in Step 2
    public List<ActionCount> countActions (List<UserActivity> userActivityList) {
	    // If input is not empty or null continue
        if (userActivityList != null && !userActivityList.isEmpty()) {
	        // Create hashmap with Action name as key and it's count
            Map<String, ActionCount> actionCountMap = new HashMap<>();
            // For every user activity in input, skip broken ones and get action name
            for (UserActivity userActivity : userActivityList) {
                if (userActivity != null && userActivity.getAction() != null) {
                    String action = userActivity.getAction();
                    // If new action create new counter and put in table
                    if (!actionCountMap.containsKey(action)) {          
                        ActionCount actionCount = new ActionCount();
                        actionCount.setAction(action);
                        actionCount.setCount(0);
                        actionCountMap.put(action, actionCount);
                    }
                    // Then incriment action
                    ActionCount actionCount = actionCountMap.get(action);
                    actionCount.incrementCount();                         // +1
                }
            }
            // Return hashmap values as array list
            return new ArrayList<>(actionCountMap.values());
        }
        return null;
    }

    // With the steps defined, we can create the template method itself
    public List<ActionCount> generateReport(String filePath) {
        try {
	        // step 1
            String data = readData(filePath);                                           // step 2 (decided by subclass)
            List<UserActivity> activityList = parseFileContent(data);                   // step 3
            List<ActionCount> actionCountList = countActions(activityList);
            return actionCountList;
        } catch (Exception e) {
            handleException(e);
        }
        return null;
    }
}
```
- ActionCount is a helper class that stores just an action string, and an integer count with getters, setters and an incrimentCount function

Subclasses can then override the gap:
```java
public class CsvActionReport extends ActionReport {       
// only fills in step 2

    @Override
    public List<UserActivity> parseFileContent(String rawData) {
        List<UserActivity> userActivityList = new ArrayList<>();
        if (rawData != null) {
	        // Create array, where each element is one seperate line
            String[] lines = rawData.split("\n");   
            // For every line, split the different cells            
            for (String line : lines) {
                String[] strings = line.split(";"); 
                // Skip blank or incorrectly formatted lines            
                if (strings != null && strings.length == 4) {
                    String username = strings[0];
                    String action   = strings[1];
                    String content  = strings[2];
                    Integer id      = Integer.parseInt(strings[3]);
                    // Add to userActivityList in correct format
                    userActivityList.add(new UserActivity(username, action, content, id));
                }
            }
        }
        return userActivityList;
    }
}
```
- All other steps remain the same, only the CSV specific one is changed. 

The same can be done with the XML subclass:

```java
import java.io.*;
import java.util.*;
import javax.xml.parsers.*;
import org.w3c.dom.*;
import org.xml.sax.*;

public class XmlActionReport extends ActionReport {
     @Override
     public List<UserActivity> parseFileContent(String rawData) {
         List<UserActivity> userActivityList = new ArrayList<>();
         try {
	         // From library
             DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
             DocumentBuilder db = dbf.newDocumentBuilder();
             // Convert string to file
             InputSource is = new InputSource(new StringReader(rawData));
             // Turn XML text into tree of objects (Document type doc)
             Document doc = db.parse(is);
             // Find outermost element
             Element root = doc.getDocumentElement();
             // Get all <action> elements inside root, store in NodeList
             NodeList nodeList = root.getElementsByTagName("action");
             // Loop through them, one at a time
             for (int i = 0; i < nodeList.getLength(); i++) {
                 org.w3c.dom.Node n = nodeList.item(i);
                 // Make sure it is of the correct type
                 if (n.getNodeType() == org.w3c.dom.Node.ELEMENT_NODE) {
                     Element elem = (Element) n;
                     // Get relevant attributes and pass to userActivity
                     String username = elem.getAttribute("usernmame");
                     String actionName = elem.getAttribute("actionName");
                     String content = elem.getAttribute("content");
                     Integer id = Integer.parseInt(elem.getAttribute("id"));
                     UserActivity userActivity = new UserActivity(username, actionName, content, id);
                     userActivityList.add(userActivity);
                 }
             }
         } catch (ParserConfigurationException e) {e.printStackTrace();
         } catch (IOException e) { e.printStackTrace();
         } catch (SAXException e) { e.printStackTrace();
         }
         return userActivityList;
     }
}
```
### Iterator 
Intent: 
- Travesing elemnts of a collection object without exposing its underlying representation (list, tree, stack, graph)
	- Give a way to go through your items one at a time, without showing them how they are stored
- Keeps client traversal code loosely coupled to the collection and its traversal algorithm.
Use When: 
- You need to access elements in a specific order (front-to-back, back-to-front, DFS/BFS on trees, skipping elements)
- You want to walk through every element of a collection _without the outside world knowing how the collection is built inside_

The Problem: 
```java
public class ProductPortfolio {
    private Product products[];

    public Product[] getProducts() {
        return products;              // hands out the real internal array
    }
}
```
A client will use this setup:
```java
static void clientCode(ProductPortfolio portfolio) {
    Product products[] = portfolio.getProducts();
    for (int i = 0; i < products.length; i++)
        process(products[i]);
}
```
- Client will be able to edit actual array because it is handed out
- If we switch from an array to an arraylist client code will break

This can be fixed with the Iterator pattern
![[file-Pasted image 20260913155202-20260913155202053.jpg|400]]
- Client only talks to the two interfaces
	- Iterator: the "librarian" interface, with hasNext() and next()
	- IterableCollection: anything that can hand you a librarian, through createIterator()
- ConcreteCollection: holds the actual items
- ConcreteIterator: walks through those items and remembers where it's up to

Creating the two interfaces:
```java
// Thing that walks through the items
public interface Iterator {
    public boolean hasNext(); // Gives true if at least one more item
    public Object next(); // Gives you the next item
    // Returns Object so works for any kind of item, just needs to be casted
}
// Thing that holds the items
public interface IterableCollection {
    public Iterator createIterator(); // Any collection that implements this must give an interator to access it to the implementer 
}

```

Now to create the collection itself
```java
public class FriendsCollection implements IterableCollection {
	// The data itself
    private String[] names = { "Friend_1", "Friend_2", "Friend_3" };
	
	// There is no getter for the array, just this iterator that can be used to walk through it
    @Override
    public Iterator createIterator() {
        return new FriendsIterator();
    }

    // inner class that builds the iterator itself: has access to `names`, outside code doesn't
    private class FriendsIterator implements Iterator {
        private int index = 0;
		
		// Simple check to see if there is a next element
        @Override
        public boolean hasNext() {
            return names != null && index < names.length;
        }
        // Return next object if it exists
        @Override
        public Object next() {
            if (this.hasNext()) {
                return names[index++];
            }
            return null;
        }
    }
}
```

This can then be used by a client implemention:
```java
public class IteratorDemo {
    public static void main(String[] args) {
        FriendsConcreteCollection f = new FriendsConcreteCollection();
		// Loop and print all data, until there is no next
		// No need for i++, since .next already does that
        for (Iterator iter = f.createIterator(); iter.hasNext(); ) {
            String name = (String) iter.next();
            System.out.println("Name : " + name);
        }
    }
}
```

- If we updated FriendsListCollection to use an ArrayList, this client code would still work. 
- Java's existing for each loop:  `for (String name : names)` is an example of this
## Structural Patterns 

- Concerned with how classes and objects are composed into larger structures
	- How are classes and objects put together to build something bigger?
- Class-scope ones combine pieces using inheritance
- Object-scope ones combine pieces by having objects hold other objects.
	- More flexible, because you can change which object is plugged in while the program runs

### DAO (Data Access Object)

Intent:
- Decouple domain logic from persistence mechanisms 
	- Keep the logic of the app (logging in, creating posts etc) separate from the things that save data
	- This keeps them separate, so one can change without breaking the other
- Used to not expose details of storage details 
	- App's logic should never know how or ever data is saved, just asked DAO to do it
Use When: 
- Access to data varies depending on source of data
- When business components need to access data, they can use an API instead of directly manipulating the data source

The Problem:
Assume our previous session state was saving posts:
```java
public void createPost(String content) {
    String text = username + ";create-post;" + content + ";" + id + "\n";
    Files.write(Paths.get("/tmp/posts.csv"), text.getBytes(), StandardOpenOption.APPEND);
}

public void likePost(Integer postId) {
    String text = username + ";like-post;+1;" + postId + "\n";
    Files.write(Paths.get("/tmp/posts.csv"), text.getBytes(), StandardOpenOption.APPEND);
}
```
- If we move away from a CSV to a database, we will have to rewrite every method
- Similarly, every class that touches data has access to alll of this information

![[file-Pasted image 20260915151119-20260915151119285.jpg|425]]
DAO solves this with three parts:
- Model (Student): a plain box of data
- DAO Interface (StudentDAO):  defines what you can do with the data (like an API)
- DAO implementation (StudentDaoImpl): how it's actually stored (logic for interface)
Rest of the app only talks to DAO Implemention

Starting with the Model:
```java
public class UserActivity {
    private String username;
    private String action;
    private String content;
    private Integer idPost;
	
	// Constructor
    public UserActivity(String username, String action, String content, Integer idPost) {
        this.username = username;
        this.action = action;
        this.idPost = idPost;
        this.content = content;
    }
    
    // Getters
    public String getUsername() { return username; }
    public String getAction()   { return action; }
    public String getContent()  { return content; }
    public Integer getIdPost()  { return idPost; }
}
```
- Every action is stored as an activity (same as the template method)

Then the DAO interface (contract of what can be done with data, no mention of how it's stored):
```java
public interface UserActivityDaoInterface {
    public UserActivity createPost(String username, String postContent);
    public UserActivity likePost(String username, Integer idPost);
    public List<UserActivity> findAllPosts();
    public String getFilePath();
    public void deleteAll();
}
```
- Defines the five things the app can ask for 

Now the DAO implementation to actually get the data for the interface methods from the storage method: 
- This assumes it's a CSV file, if we switch to another type, all we have to do is update this class without needing to change client implementions
```java
public class UserActivityDao implements UserActivityDaoInterface { 
	// All static so one copy for whole program  
    private static UserActivityDao instance; // singleton
    private static File file;
    private static Integer idCount = 0;
	
	// Runs only once at start of program, read singleton for more details
    static {                                        
        try {
            file = File.createTempFile("user-action", ".csv");   
        } catch (IOException e) { e.printStackTrace(); }
    }
	
	// It's a singleton, so a private constructor
    private UserActivityDao() {}         

	// Intialize singleton
    public static UserActivityDao getInstance() {
        if (instance == null) instance = new UserActivityDao();
        return instance;
    }
```
- We initialise as a singleton because everyone in the app shares one DAO and one file

Now for the class implementations itself:
```java
    public UserActivity createPost(String username, String postContent) {
        try {
	        // get new unique post id
            idCount++;  
            String action = "create-post";
            // Build line in CSV format and append to file
            String text = username + ";" + action + ";" + postContent + ";" + idCount + "\n";
            Files.write(file.toPath(), text.getBytes(), StandardOpenOption.APPEND); 
            System.out.println("Post saved in " + file.getAbsolutePath());
            // Create in userActivity format and return to caller
            UserActivity userActivity = new UserActivity(username, action, postContent, idCount); 
            return userActivity;
        } catch (IOException e) { e.printStackTrace(); }
        // Return error and null if failed
        return null;
    }
	
	// Same logic for liking
    public UserActivity likePost(String username, Integer idPost) {
        try {
            String action = "like-post";
            String content = "+1";
            String text = username + ";" + action + ";" + content + ";" + idPost + "\n";
            Files.write(file.toPath(), text.getBytes(), StandardOpenOption.APPEND);
            System.out.println("Like saved in " + file.getAbsolutePath());
            return new UserActivity(username, action, content, idPost);
        } catch (IOException e) { e.printStackTrace(); }
        return null;
    }
	
	// Getting all posts in UserActivity list
    public List<UserActivity> findAllPosts() {
        List<UserActivity> userActivityList = new ArrayList<>();
        try {
	        // Read whole file, split into lines
            byte[] bytes = Files.readAllBytes(file.toPath());
            String[] lines = new String(bytes).split("\n");
            // For each line, split into the columns
            for (String line : lines) {
                String[] s = line.split(";");
                if (s != null && s.length == 4) {
	                // Only return posts, not likes
                    if ("create-post".equals(s[1])) {               
                        userActivityList.add(new UserActivity(s[0], s[1], s[2], Integer.parseInt(s[3])));
                    }
                }
            }
        } catch (IOException e) { e.printStackTrace(); }
        return userActivityList;
    }

    public String getFilePath() { return file.getAbsolutePath(); }

    public void deleteAll() {
        try {
            if (file.exists()) file.delete();
            // Replace with new file
            file = File.createTempFile("user-action", ".csv");  
        } catch (IOException e) { e.printStackTrace(); }
    }
}
```

This can then be used by a client:
```java
UserActivityDaoInterface dao = UserActivityDao.getInstance();
dao.createPost("admin", "hello");
dao.createPost("bob", "hi");
dao.likePost("admin", 1);
List<UserActivity> posts = dao.findAllPosts();
```
### Facade 

Intent:
- Provide a unified higher-level interface to a set of interfaces in a subsystem, making it easier to use 
	- Subsystem: group of classes that work together on something
	- Give a client a way to ask for what they want (create a post), instead of having to call on multiple classes in the right order ("log in, post, make a report, log out")
- Just one class, no use of inheritance or polymorphism
Use When: 
- Need a single point of access for clients to simplify code 
- You want to offer abstraction to client classes and decrease coupling (clients are buffered from subsystem changes)
![[file-Pasted image 20260917195314-20260917195314218.jpg|400]]
- DAO hides persistence, Facade hides any complex subsystem behind an even simpler interface. So you can have Facade on top of DAO. 

The Problem: 
- If a client wants to create a post, they have to do the following:
```java
UserSession userSession = new UserSession();
userSession.login("admin", "123");
UserActivity post = userSession.createPost("abc");
List<ActionCount> report = userSession.generateActionCountReport();
userSession.logout();
```
- five steps, in the right order, with three different classes to know about

The facade class: 
- One method that takes everything for the needed job, returns a UserDTO that bundles the result
```java
// wraps complex operations
public class UserFacade {
	// Facade method for creating post
    public UserDto createPost(String username, String password, String content) {
	    // Create user session, login and create post
        UserSession userSession = new UserSession();
        userSession.login(username, password);
        UserActivity userActivity = userSession.createPost(content);
        // Build a report using CsvActionReport (from Template Method)
        List<ActionCount> actionCountList = userSession.generateActionCountReport();
        // Log out
        userSession.logout();
        // Bundle new post and report into one object and return
        UserDto userDto = new UserDto(userActivity, actionCountList);
        return userDto;
    }
    // Facade method for Liking all posts
    public UserDto likeAllPosts(String username, String password) {
	    // Login and find posts
        UserSession userSession = new UserSession();
        userSession.login(username, password);
        List<UserActivity> allPosts = userSession.findAllPosts();
        UserActivity lastUserActivity = null;
        // If there are any posts, loop and like each one
        if (allPosts != null && !allPosts.isEmpty()) {
            for (UserActivity userActivity : allPosts) {
                userSession.likePost(userActivity.getIdPost());
                lastUserActivity = userActivity;
            }
        }
        // Generate report
        List<ActionCount> actionCountList = userSession.generateActionCountReport();
        userSession.logout();
        // Bundle last liked post, and report in one object and return
        UserDto userDto = new UserDto(lastUserActivity, actionCountList);
        return userDto;
    }
}
```

The DTO is a Data Transfer Object
- Simply a box/data class for carrying data back to caller
- Needed as return can only return one thing, but facade has two results: post and report, so it gets put in one box (like a tuple)
- No logic, just fields and getters
```java
public class UserDto {
    private UserActivity lastPostCreated;
    private List<ActionCount> communityReport;

    public UserDto(UserActivity lastPostCreated, List<ActionCount> communityReport) {
        this.lastPostCreated = lastPostCreated;
        this.communityReport = communityReport;
    }
    public UserActivity getLastPostCreated() { 
	    return lastPostCreated; 
    }
    public List<ActionCount> getCommunityReport() { 
	    return communityReport; 
    }
}
```

This can be used by the client:
```java
public static void main(String[] args) {
    UserFacade userFacade = new UserFacade();
    userFacade.createPost("admin", "123", "abc");
    userFacade.createPost("admin", "123", "abcd");
    UserDto userDto = userFacade.createPost("admin", "123", "abcf");
    System.out.println(userDto);
}
```
- Three posts in one line, instead of multiple lines for one post
- main never mentions underlying classes, only knows UserFacade and UserDto
- Primary drawback: with a wrong password, you just get a null since we have no error checking, this can be added. 

### Decorator 

Intent: 
- Dynamically add or modify the behaviour of an object at runtime without changing its original structure.
	- Allows us to wrap an object with additional functionality while keeping the same interface
Use When: 
- Add behaviour at runtime
	- Decide what extras an object gets, while the program runs
- Avoid a explosion of subclasses
	- OrangeJuice, OrangeJuiceWithSugar, OrangeJuiceWithSugarAndCarrot
- Keep code open for extension but closed for modification
	- Nothing that already works gets edited 
- Add responsibilities to individual objects, not a whole class
	- Give one specific instance of an object something extra, not every instance of that object

The Problem:
```java
class CheesePizza extends PlainPizza { ... }
class PepperoniPizza extends PlainPizza { ... }
class CheesePepperoniPizza extends PlainPizza { ... }
class CheesePepperoniMushroomPizza extends PlainPizza { ... }
// ...one class for every combination
```
- With normal inheritance, we have too many classes 

![[file-Pasted image 20261007033641-20261007033641202.jpg|325]]
![[file-Pasted image 20261007033651-20261007033651042.jpg|500]]
- Pizza (component interface) is implemented by PlainPizza (concrete component) and PizzaDecorator (abstract decorator interface)
- PizzaDecocrator is a Pizza and has a Pizza

Common Interface Pizza:
```java
public interface Pizza {
    String getDescription();
    double cost();
}
```
- Everything that counts as "a pizza", whether plain or with any toppings, must be able to describe itself and say its price.

The base pizza: 
```java
public class PlainPizza implements Pizza {
    @Override
    public String getDescription() {
        return "Plain pizza";
    }

    @Override
    public double cost() {
        return 8.0; // Base price of the pizza
    }
}
```
- Innermost layer, doesn't wrap anything, just gives a description and base price 

Abstract Wrapper, PizzaDecorator:
```java
// Implements Pizza, so anywhere you use Pizza, you can use a decorated one
public abstract class PizzaDecorator implements Pizza { 
	// Decorator also has a pizza that it's wrapping (layer under it)
	// This means it can wrap either a Plain Pizza or a another decorator
	// This is what let's us stack layers
    protected Pizza decoratedPizza;                        
	
	// Constructor that sets the underlying Pizza
    public PizzaDecorator(Pizza decoratedPizza) {
        this.decoratedPizza = decoratedPizza;
    }
    // Default functions that get values from the inner pizza and return answer unchanged, will be overriden by toppings
    @Override public String getDescription() { 
	    return decoratedPizza.getDescription(); 
    }  
    @Override public double cost() { 
    return decoratedPizza.cost(); 
    }
}

```

A topping, CheeseDecorator: 
```java
public class CheeseDecorator extends PizzaDecorator {
	// Pass the inner layer and store in decoratedPizza
    public CheeseDecorator(Pizza decoratedPizza) {
        super(decoratedPizza);
    }
    // Ask inner pizza for values, and add own values to end
    @Override
    public String getDescription() {
        return decoratedPizza.getDescription() + ", cheese";
    }
    @Override
    public double cost() {
        return decoratedPizza.cost() + 1.5; // Cost of cheese topping
    }
}
```
This can be replicated for other toppings:
```java
public class PepperoniDecorator extends PizzaDecorator {
    public PepperoniDecorator(Pizza decoratedPizza) {
        super(decoratedPizza);
    }

    @Override
    public String getDescription() {
        return decoratedPizza.getDescription() + ", pepperoni";
    }

    @Override
    public double cost() {
        return decoratedPizza.cost() + 2.0; // Cost of pepperoni topping
    }
}
```

The client can then use this, in class PizzaShop:
```java
public class PizzaShop {
    public static void main(String[] args) {
        Pizza pizza = new PlainPizza();
        // Plain pizza $8.0
        System.out.println(pizza.getDescription() + " $" + pizza.cost()); 
        // Wrap cheese around the current pizza, right side of this runs first so it adds cheese to the current pizza and sets the same variable to the updated pizza with cheese   
        pizza = new CheeseDecorator(pizza);                                 
        System.out.println(pizza.getDescription() + " $" + pizza.cost());   // Plain pizza, cheese $9.5
        pizza = new PepperoniDecorator(pizza);                              
        System.out.println(pizza.getDescription() + " $" + pizza.cost());   // Plain pizza, cheese, pepperoni $11.5
    }

This creates:
┌──────────── PepperoniDecorator ─────────────┐
│  ┌──────── CheeseDecorator ──────────┐      │
│  │  ┌──── PlainPizza ────┐           │      │
│  │  │ "Plain pizza", 8.0 │           │      │
│  │  └────────────────────┘           │      │
│  └───────────────────────────────────┘      │
└─────────────────────────────────────────────┘
```
- You can also add doublecheese etc

![[file-Pasted image 20261007035515-20261007035515921.jpg|550]]