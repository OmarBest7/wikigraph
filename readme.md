wikiGraph
===========
What connects two topics on Wikipedia? For example how many links do you have to click to get from Harry Potter to the Spanish Inquisition?* Combining trivia nerdery with graph theory, wikiGraph allows users to find and explore the paths within Wikipedia.

You can check out the project in progress [here](http://ec2-54-148-235-143.us-west-2.compute.amazonaws.com/).

*It takes 3 clicks: Harry Potter -> British literature -> the spread of the printing press -> the Spanish Inquisition

###Contents
- [Features](#features)
- [The graphs](#the-graphs)
- [Queries](#queries)
- [Data visualization](#data-visualization)
- [User input](#user-input)
- [Improving response time](#improving-response-time)
- [Deployment](#deployment)

#### Features
*Current*
- [x] Wikipedia page links imported into a graph database (Neo4j)
- [x] Python wrapper queries database for shortest path between two nodes, outputs path and secondary relationships as JSON (py2neo)
- [x] Subgraph rendered as a force-directed graph (d3.js)
- [x] Flask app renders html and handles AJAX requests to the database
- [x] Flask app and database deployed (EC2, Apache)
- [x] Search suggest for page titles (typeahead.js, SQLite)
- [x] Embed page images on nodes within the rendered graph (Wikipedia API)
- [x] Option to generate a random search query

*Future*
- [ ] Nodes are sized/colored based on the number of links to other nodes
- [ ] Incorporate summary of path pages as mouseover tooltips (Wikipedia API)
- [ ] Path responses cached (CouchDB)

#### The graphs
I downloaded RDF files (.ttl) for page links and redirects from [DBPedia](http://wiki.dbpedia.org/Downloads2014). Here's what the raw page links file looks like:
```
<http://dbpedia.org/resource/Anarchism> <http://dbpedia.org/ontology/wikiPageWikiLink> <http://dbpedia.org/resource/William_McKinley> .
<http://dbpedia.org/resource/Alabama> <http://dbpedia.org/ontology/wikiPageWikiLink> <http://dbpedia.org/resource/Andrew_Jackson> .
```
Since the links file was quite large, I ran <kbd>clean_ttl.py</kbd> to just pull out the page names from the source and target and write them to a tab-separated file, since I didn't need the url or the link between them. This significantly reduced the file sizes for both links and redirects (23GB -> 6.2GB, 980MB -> 275MB).

I then used a Python script to parse and clean the page links. At first this meant removing redirect pages and duplicates. After looking at the output, I realized that several types of pages were only ever found as target pages and never appeared as sources, e.g. Files, Categories, and Help. Additionally, even outside those categories, almost half of the pages in the file never linked to anything else--they were dead-ends. Most are [red links](http://en.wikipedia.org/wiki/Wikipedia:Red_link) which are links that point to a nonexistent page (and some I'm absolutely baffled by).

Here is the main function in <kbd>master_clean.py</kbd>:

```python
def clean_data():
    """Creates a tsv file for page links and one for pages. First it assembles 
    a dictionary of redirect pages, then uses that to create a dictionary of 
    deduped page links. It then parses the dictionary of links for pages that are not dead ends, and writes their nodes and relationships to the two files."""

    redirects = redirects_dict('data/cleaned_redirects.tsv')
    data = assemble_dict('data/cleaned_links.tsv', redirects)
    codes = recode_dict(data)
    pruned_data = prune_data(data, codes)
    write_rels(pruned_data, 'data/rels.tsv')
    write_nodes(pruned_data, 'data/nodes.tsv')
```
It assembles a dictionary of all the information within the raw data file. Degrees are calculated and added during the pruning phase, after which the page links dictionary now looks like this:
```python
{'page1': {'code': 41, 'title': 'page1', 'degrees': 3, links': set([42, 108, 109])}}
```
Wikipedia is big! The raw data include over 152 million relationships. After cleaning, the complete graph has over 4 million nodes (160MB) and 113 million edges (2.6GB). The data are stored in two tsv files: a list of all relationships (*start, end*) and a list of all nodes (*node, name, label, degrees*).

__nodes.tsv__
```
node    name            l:label    degrees
0       Alabama         Pages      83
1       Andrew Jackson  Pages      51
```
__rels.tsv__
```
start   end type
0       1   LINKS_TO
2       3   LINKS_TO
```
I used Michael Hunger's [batch import tool](https://github.com/jexp/batch-import/tree/20) to insert the data into a [Neo4j](http://neo4j.com/) graph database. Then, I applied a constraint on all nodes that their id ('node') was unique (using Neo4j's browser interface).
```
CREATE CONSTRAINT ON (p:Page) ASSERT p.node IS UNIQUE;
```

At this point, after some initial queries, I realized that a responsive query of such a large database would take some refinement (see [Improving query response time](#improving-query-response-time) below) and I wanted to figure out how to display my data first. I wrote <kbd>pres_clean.py</kbd> to sample the pagelinks file for only those pages and links that include the names of U.S. Presidents. After cleaning, this graph had 77 thousand nodes and 137 thousand relationships. All of my initial testing and design used this subgraph until I could decrease the response time.

Complete graph | Subgraph
-------------- | -----------
11m nodes | 77k nodes 
127m links | 137k links

#### Queries
I used Nigel Small's Python library [py2neo](http://nigelsmall.com/py2neo/1.6/) to interact with Neo4j's RESTful web service interface. <kbd>query.py</kbd> translates my shortest-path request into a CypherQuery object, queries the database, and returns the results as a Path object. 
```python
query = neo4j.CypherQuery(
    graph_db, 
    """MATCH (m:Page {node:{n1}}), (n:Page {node:{n2}}), 
    p = shortestPath((m)-[*..20]->(n)) RETURN p"""
)
query.execute(n1=node1, n2=node2)
```
The script then traverses this path object, pulling out and deduping nodes and relationships. The ids need to be recoded to be sequential, starting from 0. Finally, the nodes and relationships are formatted and returned as JSON.
```
{
    "directed": true,
    "nodes": [
        {
            "degrees": 22,
            "node": 0,
            "name": "William Persse",
            "group": "path"
        },
        {
            "degrees": 102,
            "node": 1,
            "name": "George Washington",
            "group": "path"
        },
        {
            "degrees": 35,
            "node": 2,
            "name": "American Presidents: Life Portraits",
            "group": "none"
        }
    ],
    "links": [
        {
            "start": 0,
            "end": 1,
            "value": 1
        },
        {
            "start": 1,
            "end": 2,
            "value": 0
        }
    ],
    "multigraph": false
}
```

#### Data visualization
<kbd>wikigraph.py</kbd> is a small [Flask](http://flask.pocoo.org/) app that connects the database's reponse to the [d3 library](http://d3js.org/). <kbd>graph.js</kbd> handles the graph drawing while <kbd>index.js</kbd> handles everything else.

Wikipedia page images are sourced from the [Wikimedia API](http://www.mediawiki.org/wiki/API:Main_page) via two AJAX requests: once for the start and end nodes upon the query request, and then for the inner path nodes once the result is received.

#### User input
To help users input page names correctly (and to suggest possible queries) I implemented a predictive seach with [typeahead.js](https://twitter.github.io/typeahead.js/). Via an AJAX call, it queries an indexed [SQLite](http://www.sqlite.org/) database that holds the page titles and their codes.

#### Improving query response time
At the start of the project, I decided there were at least four possible approaches to improve response time. I've tackled three of them so far, and I've seen improvements with each:
- [x] Scale vertically (tweak memory allocation, use larger machine)
- [x] More efficient query (change query parameters, possibly rewrite algorithm)
- [x] Prune graph if possible (remove trailing linked tails?)
- [ ] Scale horizontally (distributed processing, e.g. [Giraph](http://giraph.apache.org/))

#####Scale vertically
My first approach to improve response time for the full database was to fiddle with Neo4j's memory settings. The settings in **neo4j.properties** (e.g. *neostore.nodestore.db.mapped_memory*) didn't have a large impact on query time. I had more success with *java.initmemory* and *java.maxmemory* (in **neo4j-wrapper.conf**).

Each time I increased both init and max memory, I ran the same query three times and recorded the response time. My MacBook Air has 4G of RAM, which seems to coincide with the dramatic improvement in query time (1400s to 60s) after passing the 4G mark. (This is odd, considering all advice I've seen suggests to leave 1-2GB for the OS, etc.)

![Memory Test Results](static/images/mem_test.png)

Then, I deployed the database to a larger machine (see [Deployment](#deployment) below). I scaled the java memory settings to the new specs, but the query time only halved (60 sec to 30 sec) despite the four-fold increase in RAM.

#####Query efficiency
I chose to use the built-in shortest-path algorithm for Neo4j, even though I've been unable to find out exactly what the algorithm is. [Here](https://groups.google.com/forum/#!topic/neo4j/GiQPwQC_rII) is the closest description I've found:

>The shortest path algorithm (i.e. paths with as few relationships as possible) uses breadth first, alternating sides between each visited relationship, not between each fully expanded depth. So if one side expands into fewer relationships than the other, that side can continue to new depths and find candidates even if the other side doesn't advance. When candidates are found the current depths will be fully expanded and shortest candidates will be selected and returned.

The good folks on the [Neo4j Google Group](https://groups.google.com/forum/#!forum/neo4j) then suggested that the lookup of the two nodes was likely the slowest factor (rather than the pathfinding algorithm). Here is my initial query:
```python
query = neo4j.CypherQuery(
    graph_db, 
    """MATCH (m {node:'%s'}), (n {node:'%s'}), 
    p = shortestPath((m)-[*..20]->(n)) RETURN p""" % (node1, node2)
)
query.execute_one()
```
I added a constraint in the database for the Page label (all nodes are Pages) to express that node id is unique:
```
CREATE CONSTRAINT ON (p:Page) ASSERT p.node IS UNIQUE;
```
And then I modified my query to use the Page label in the node lookup, as well as pass the nodes as arguments (instead of via string substitution):
```python
query = neo4j.CypherQuery(
    graph_db, 
    """MATCH (m:Page {node:{n1}}), (n:Page {node:{n2}}), 
    p = shortestPath((m)-[*..20]->(n)) RETURN p"""
)
query.execute(n1=node1, n2=node2)
```
Surprisingly, auto-indexing had no effect on this query. I'd had it turned on (and assigned to index on 'node', e.g. id), but it was not adding efficiency. The constraint added via the Page label increased the speed with which the database finds the two nodes.

##### Pruning the graph
I did not look at the data closely enough upon first inserting into the database, and failed to notice the large proportion of 'dead-end' nodes: nodes that either didn't exist on Wikipedia (although they were linked to) or they'd been deleted, etc. I was happy to remove them, since I couldn't include those nodes in a query to find the shortest path! By eliminating dead-ends, my graph is now fully connected--and half the size I thought it would be.

#### Deployment
This code was tested on Amazon's [EC2](http://aws.amazon.com/ec2/) using [Apache](http://httpd.apache.org/) as a web server. The database is housed on a 30 GiB EBS. Currently it is on an r3.large server with 15G RAM, and the query of the full database takes just 0.5 seconds. Since EC2 servers do not come with virtual memory, I set up the 32G SSD ephemeral instance storage as a paging (or swap) partition to give the database access if needed.