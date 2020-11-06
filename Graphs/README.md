# Dealing with Graphs

**TODO**: Need to split this (and other sections) into "using recursion schemes" and "understanding recursion schemes" somehow.

So, let's talk about how we want to deal with graphs ... first, we'd like to maintain as much as possible the same abstractions we use when dealing with trees. That is, algebras and pattern functors. This means that given the same pattern functor, we could build either a tree or a graph (just as we can build either a finite or infinite structure from the same pattern functor). And we'd like to be able to apply the same algebras (but maybe not get the same results -- we'll come back to that point).

So, we have our standard pattern functors, `f a`, but our usual way of working with them (for trees) involves replacing the `a` with a "subtree", e.g., `Fix f`. The problem here is that the direct reference means that we can only refer to each node from one place -- the parent that contains it. But, pattern functors are pretty flexible, so we can instead have a bunch of nodes that instead of the `a` being a node itself is a _reference_ to some node. Most simply, `[f Int]`, where each Int is an index for some other node in the list. This is called an "adjacency list". We can ensure that we have a DAG by checking that we can reorder the list such that each reference is strictly `<` the current index (and the graph construction functions we introduce later all build graphs already ordered that way).

Additionally, in order to fold a graph, we need some way to indicate which node we care about the result for. This can be managed externally, but we can also have what's called a "rooted graph", where one or more nodes is blessed as a "root" node, giving us some way to extract the values we care about. We can model these roots fairly abstractly, as `Traversable r => r Int`, where `r` is often `Identity` (for a singly-rooted graph) or `[]` (for a multi-rooted graph), and the `Int`s point to the index of the root nodes.

```haskell
-- | A graph with vertices of type `v ()` and some structure `r` of roots.
newtype Graph v = Graph 
  { -- | Each entry in the list is a vertex in the graph. The `Int`s represent
    --   edges from the current vertex to the vertex at the index indicated by
	--   the `Int`.
    vertices :: [v Int],
  }
  deriving (Generic)

data MultiplyRootedGraph r v = MultiplyRootedGraph
  { graph :: Graph v,
    -- | The roots of the graph are some structure (often `Identity` or `[]`)
	--   indicating vertices that are "blessed" as some starting point of the
	--   graph. If the graph was constructed from `eliminateCommonSubtrees`,
	--   each root corresponds to the root of one of the trees passed to that
	--   function.
    roots :: r Int
  }
  deriving (Generic)

type RootedGraph v = MultiplyRootedGraph Identity v
```

We can then take advantage of our usual `f a -> a` algebras by traversing the list of vertices, building a parallel list with the `a` in place of the `f Int`, and looking up the indices of the children as we go. If our graph is rooted, we finish by looking up the `a` for each root index.

## Implementation

```haskell
-- | A graph with vertices of type `v ()` and some structure `r` of roots.
data Graph r v
  = Graph
      { -- | Each entry in the list is a vertex in the graph. The `Int`s
	    --   represent edges from the current vertex to the vertex at the index
		--   indicated by the `Int`.
        graphVertices :: [v Int],
        -- | The roots of the graph are some structure (often `Identity` or
		--  `[]`) indicating vertices that are "blessed" as some starting point
		--   of the graph. If the graph was constructed from
		--  `eliminateCommonSubtrees`, each root corresponds to the root of one
		--   of the trees passed to that function.
        graphRoots :: r Int
      }
  deriving (Generic)

-- | Used internally to hold information we need while constructing a graph from
--   trees.
data GraphState f
  = GraphState
      { -- | These are nodes we've already come across, so we can simply return
	    --   the existing `Int` index rather than introducing a new node in the
		--   graph. This /could/ be satisfied by using
		--   @`flip` `elemIndex` . `vertices`@, but that would be O(n), whereas
		--   this is O(1).
        seenNodes :: HMap.HashMap (f Int) Int,
        -- | These are exactly the vertices (in reverse order) that will be in
		--   the graph after creation.
        vertices :: [f Int],
        -- | Another optimization, this could be `length . vertices`, but that
	    --   is again O(n) whereas this is O(1).
        lastIndex :: Int
      }

-- | This builds a single graph from some structure of trees. Any common
--   substructures in different trees will be shared the same way as common
--   substructures within a tree.
eliminateCommonSubtrees ::
  ( Traversable r,
    Recursive (->) t f,
    Traversable f,
    Eq (f Int),
    Hashable (f Int)
  ) =>
  r t ->
  Graph r f
eliminateCommonSubtrees =
  uncurry (flip Graph)
    . fmap (reverse . vertices)
    . flip State.runState (GraphState HMap.empty [] 0)
    . traverse (cataM findOrAddNode)

findOrAddNode ::
  (Eq (f Int), Hashable (f Int)) =>
  f Int ->
  State.State (GraphState f) Int
findOrAddNode tree = do
  gs <- State.get
  let seen = seenNodes gs
  case HMap.lookup tree seen of
    Just index -> pure index
    Nothing -> do
      let index = lastIndex gs
      State.put
        . GraphState (HMap.insert tree index seen) (tree : vertices gs)
        $ succ index
      pure index
```

## Benefits

One of the reasons to generate a graph is that you eliminate duplication. If your structure has many common substructures, then with a graph, you are guaranteed to only process each common structure once, no matter how many times it is referenced.
Variations

## More efficient building

We've seen how we can build a graph using a fold from a tree, but one shortcoming is that it examines every node in the tree, and the whole point of the graph is to avoid dealing with duplicated things. So, can we build a graph that can quickly identify that it doesn't need to re-explore a subtree that it's already added? Yes, with some caveats ...
1. this isn't total
2. it might not identify all identical substructures (although I haven't yet seen it fall short in practice)
3. it's not a simple fold

## Avoiding building a graph in the first place

Something we've already explored with trees is how we can gain some "fusion" by combining multiple algebras. E.g., `zygo` lets us apply a "helper" algebra alongside the primary algebra rather than first annotating a tree then folding the annotated tree.

Similarly, we can get some fusion in graphs by modifying the `eliminateCommonSubstructure` builder to apply an algebra immediately rather than simply creating and looking up indices. That is, we can apply an algebra to a tree in such a way that it treats the tree as if it were a graph. And this can be done with either the total fold or with the more efficient building of a graph.

## What about infinite graphs?

So far, we've only discussed building graphs from `Recursive` structures, which guarantees us a finite graph.

There are two ways we could potentially end up with infinite graphs. One is that the graph has cycles, which means there are actually a finite number of nodes, but nodes can refer back to other nodes we've already seen, forming a loop. These graphs can be generated in a finite way, but not folded without partiality.

The other way is just to have a graph that keeps growing, much like an infinite tree, not necessarily with cycles. Not only can this graph not be folded, it also can't be generated finitely.

## Not getting the same results.

If you have a DAG (directed acyclic graph), you can think of it roughly as a tree where any subtree that exists in multiple places is only "stored" once. The difference (beyond just efficiency of processing) appears once you want to fold the graph with some monadic context. With trees, we've seen that a monadic fold is simply a fold that distributes the monad across the functor before calling the algebra
```haskell
cataM phi = cata (phi <=< sequenceA)
```
so `sequenceA` converts `f (m a)` to `m (f a)`, then calls the algebra on the `f a` and finally joins the resulting `m (m a)` into `m a`.

If you were to apply this same `phi <=< sequenceA` algebra to a graph, you would get exactly the same behavior as applying it to a tree.

So what's the problem?

The problem is that we're not treating the graph like a graph. Let's look at the type of that algebra `f (m a) -> m a`. Earlier we discussed how each node in a graph is only processed once, no matter how many times it's referenced. But, in this case, what is the result of processing that node? It's an `m a`.

Basically, even though we only generate a single term of type m a for each node, that node may have it's monadic effect evaluated multiple times (once for each node it's a child of). But, we should expect each node in a graph to have its effect evaluated exactly once. Imagine if the monad is some `State` that attaches an identifier to each node in the graph. When you evaluate that `State` twice, you end up attaching two different identifiers to the same node, depending on where it's referenced from, and that's just not going to work. You might think "well, I can have the `State` track whether we've already seen this node and, if so, return the same identifier," but
1. that's just overhead when you apply the same algebra to a tree and
2. that's exactly the sort of benefit you get from using a graph in the first place, so why duplicate it?

It turns out  that we can define a (somewhat more complicated) `cataM` for graphs that accepts the same algebras as the `cataM` for trees, but without triggering multiple effect-evaluations per node. The takeaway here is that you should use `cataM` directly rather than relying on `cata . (<=< sequenceA)` to behave correctly.

## Losing sharing 

We just talked about the correct way to preserve sharing of effects, but there is another place to be careful of losing sharing -- within the algebra itself.

If you have an algebra like
```haskell
-- | Collect all the nodes in a flat list with no relationships
nodes :: f [f ()] -> [f ()]
nodes fa = void fa : fold fa
```
and apply it to a graph, you will again get the same result as applying it to a tree. But again, this means that you have included the same node multiple times in the result list. Yeah, each node results in one `[f ()]`, but each node that points to that node includes that `[f ()]` (via the call to `fold`) in its own result, and if any of those nodes is shared ... you duplicate them again.

This is a harder problem to avoid, and I haven't come up with an approach that an algebra-writer doesn't have to take into account. Here are some of the workarounds

### don't accumulate across your algebras

This is something that's very tempting when writing algebras for trees. But it causes duplication when applied to graphs. Instead, try using a proto-algebra like `void :: f a -> f ()`. On its own, this looks less useful, but once you know you're dealing with a tree, you can do
```haskell
-- | Take an algebra that ignores its carrier and turn it into one that accumulates the results.
collect :: (Foldable f, Semigroup b) => (forall a. f a -> b) -> f b -> b
collect phi fa = foldr (<>) (phi fa) fa

nodes :: f [f ()] -> [f ()]
nodes = collect $ pure . void
```
but with a graph, you can use void directly and extract the vertices
```haskell
fst . nodeCata void :: Graph f -> [f ()]
```
this is precisely why graphs have additional operations that expose the vertices.

`bmap` across the structure when possible [In Haskell, there is a library called "barbies" that allows functors of this shape, and it calls the corresponding function `bmap`, so we use that name here as well]. `Recursive` structures offer a functor (not a `Functor`) that allows you to `bmap` a natural transformation on the nodes over the entire structure. So, if you can define your operation to match `forall a. f a -> g a`, you can get a `Graph f -> Graph g`.

With trees, you might have tried something like `cata (embed . myNT)` to do the same thing, but in this case, it turns out that using `bmap` on the tree is often much more efficent than that `cata`, so this is good advice to take in general.
