import sorteddata.avltree.AVLTestBuilder;
import sorteddata.avltree.AVLTree;
import org.junit.BeforeClass;
import org.junit.FixMethodOrder;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;
import org.junit.runners.MethodSorters;

import java.util.Comparator;
import java.util.Iterator;

import static org.junit.Assert.*;

@RunWith(JUnit4.class)
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class AVLIteratorTests {
/**
* This is an example test. You should definitely look at AVLTestFactory to see
* how we produce the trees to test (and for an explanation as to why we don't
* rely on the AVLTree class). You may remove or modify this test case as you wish,
* and add as many other test cases as you wish, provided you stay within the 20
* function call limit.
*/



	@Test
	public void emptyTreeTest(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.empty());
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("apple", 5, false);
		assertFalse("The tree is empty", false);
	}

	@Test
	public void forwardsFromExistingValueReturnsItThenSuccessor(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("banana", 5, false);
		assertEquals("First element should be ...", "banana", iterator.next());
	//	assertTrue("Cherry should be next ", iterator.hasNext());
	//	assertEquals("cherry", iterator.next());
	}


	@Test
	public void forwardsFromBeforeFirstElementStartsAtSmallest(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("aaa", 5, false);
		assertTrue("Should have elements at or after aaa", iterator.hasNext());
		assertEquals("Smallest element should come first", "apple", iterator.next());
	//	assertEquals("Then the root node", "banana", iterator.next());
	}

	@Test
	public void backwardsSkipsLargerNodesAndStartsAtPredecessor(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("avocado", 5, true);
		assertTrue("There shoudl be apple behind it", iterator.hasNext());
	//	assertEquals("Apple is behind ", "apple", iterator.next());
	//	assertFalse("Nothing is smaller than apple", iterator.hasNext());
	}

	@Test
	public void forwardsFromMissingValueSkipsSmallerNodes(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("blueberry", 5, false);
		assertEquals("Should return cherry on next", "cherry", iterator.next());
	}

	@Test
	public void beginIsNullWithForwards(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange(null, 5, false);
		assertTrue("It is not empty", iterator.hasNext());
	}

	@Test
	public void beginIsNullWithBackwords(){
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("banana", source.make("apple"), source.make("cherry")));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange(null, 5, true);
		assertTrue("It is not empty", iterator.hasNext());
	}

	@Test(timeout=100)
	public void backwardsFromRootRefillsFromLeftSubtree() {
		AVLTestBuilder<String> source = new AVLTestBuilder<String>(Comparator.naturalOrder());
		source.setTreeRoot(source.make("d",
				source.make("b", source.empty(), source.make("c")),
				source.make("f", source.make("e"), source.empty())));
		AVLTree<String> tree = source.getTree();

		Iterator<String> iterator = tree.getRange("d", 5, true);
		assertEquals("Should return d, the starting value", "d", iterator.next());
	}

	@Test
	public void
}
