# Search Engine System Design

## Tổng quan hệ thống

### Yêu cầu chức năng
- **Web crawling**: Thu thập nội dung từ web
- **Indexing**: Tạo chỉ mục cho nội dung
- **Search queries**: Xử lý truy vấn tìm kiếm
- **Ranking**: Sắp xếp kết quả theo relevance
- **Auto-complete**: Gợi ý khi gõ
- **Filtering**: Lọc kết quả theo category, date, type

### Yêu cầu phi chức năng
- **Scale**: 1B+ documents, 100M+ queries/day
- **Latency**: < 100ms search response time
- **Availability**: 99.9% uptime
- **Freshness**: New content indexed trong 24h
- **Relevance**: High-quality search results

## Kiến trúc hệ thống

### High-Level Architecture
```
Web → Crawler → Document Processor → Indexer → Search Index
                                              ↓
User Query → Query Processor → Ranking Engine → Results
```

### Core Components

#### 1. Web Crawler
```python
import asyncio
import aiohttp
from urllib.parse import urljoin, urlparse
from bs4 import BeautifulSoup
import robots_txt_parser

class WebCrawler:
    def __init__(self, max_concurrent=100):
        self.max_concurrent = max_concurrent
        self.visited_urls = set()
        self.url_queue = asyncio.Queue()
        self.robots_cache = {}
        self.session = None
    
    async def crawl(self, seed_urls):
        # Initialize session
        connector = aiohttp.TCPConnector(limit=self.max_concurrent)
        self.session = aiohttp.ClientSession(connector=connector)
        
        # Add seed URLs to queue
        for url in seed_urls:
            await self.url_queue.put(url)
        
        # Create worker tasks
        tasks = []
        for _ in range(self.max_concurrent):
            task = asyncio.create_task(self.crawler_worker())
            tasks.append(task)
        
        # Wait for completion
        await self.url_queue.join()
        
        # Cancel workers
        for task in tasks:
            task.cancel()
        
        await self.session.close()
    
    async def crawler_worker(self):
        while True:
            try:
                url = await self.url_queue.get()
                await self.crawl_url(url)
                self.url_queue.task_done()
            except asyncio.CancelledError:
                break
    
    async def crawl_url(self, url):
        if url in self.visited_urls:
            return
        
        if not await self.is_allowed_by_robots(url):
            return
        
        try:
            async with self.session.get(url, timeout=10) as response:
                if response.status == 200:
                    content = await response.text()
                    document = self.parse_document(url, content)
                    
                    # Send to document processor
                    await self.send_to_processor(document)
                    
                    # Extract links for further crawling
                    links = self.extract_links(url, content)
                    for link in links:
                        if link not in self.visited_urls:
                            await self.url_queue.put(link)
                    
                    self.visited_urls.add(url)
                    
        except Exception as e:
            print(f"Error crawling {url}: {e}")
    
    async def is_allowed_by_robots(self, url):
        domain = urlparse(url).netloc
        
        if domain not in self.robots_cache:
            robots_url = f"http://{domain}/robots.txt"
            try:
                async with self.session.get(robots_url) as response:
                    if response.status == 200:
                        robots_content = await response.text()
                        self.robots_cache[domain] = robots_txt_parser.RobotsTxtParser(robots_content)
                    else:
                        self.robots_cache[domain] = None
            except:
                self.robots_cache[domain] = None
        
        robots = self.robots_cache[domain]
        if robots:
            return robots.can_fetch('*', url)
        return True
    
    def parse_document(self, url, content):
        soup = BeautifulSoup(content, 'html.parser')
        
        # Extract title
        title_tag = soup.find('title')
        title = title_tag.text.strip() if title_tag else ""
        
        # Extract meta description
        meta_desc = soup.find('meta', attrs={'name': 'description'})
        description = meta_desc.get('content', '') if meta_desc else ""
        
        # Extract main content
        main_content = self.extract_main_content(soup)
        
        return {
            'url': url,
            'title': title,
            'description': description,
            'content': main_content,
            'timestamp': datetime.now().isoformat()
        }
    
    def extract_main_content(self, soup):
        # Remove script and style elements
        for script in soup(["script", "style"]):
            script.decompose()
        
        # Try to find main content areas
        main_content = soup.find('main') or soup.find('article') or soup.find('div', class_='content')
        
        if main_content:
            return main_content.get_text()
        else:
            return soup.get_text()
    
    def extract_links(self, base_url, content):
        soup = BeautifulSoup(content, 'html.parser')
        links = []
        
        for link in soup.find_all('a', href=True):
            href = link['href']
            full_url = urljoin(base_url, href)
            
            # Only crawl HTTP/HTTPS links
            if full_url.startswith(('http://', 'https://')):
                links.append(full_url)
        
        return links
```

#### 2. Document Processor
```python
import re
import nltk
from nltk.tokenize import word_tokenize
from nltk.corpus import stopwords
from nltk.stem import PorterStemmer
import spacy

class DocumentProcessor:
    def __init__(self):
        self.nlp = spacy.load('en_core_web_sm')
        self.stemmer = PorterStemmer()
        self.stop_words = set(stopwords.words('english'))
    
    def process_document(self, document):
        # Clean and normalize text
        cleaned_content = self.clean_text(document['content'])
        
        # Extract entities
        entities = self.extract_entities(cleaned_content)
        
        # Tokenize and process
        tokens = self.tokenize_and_process(cleaned_content)
        
        # Calculate document features
        features = self.calculate_features(document, tokens)
        
        processed_doc = {
            'url': document['url'],
            'title': document['title'],
            'description': document['description'],
            'content': cleaned_content,
            'tokens': tokens,
            'entities': entities,
            'features': features,
            'timestamp': document['timestamp']
        }
        
        return processed_doc
    
    def clean_text(self, text):
        # Remove HTML tags
        text = re.sub(r'<[^>]+>', '', text)
        
        # Remove extra whitespace
        text = re.sub(r'\s+', ' ', text)
        
        # Remove special characters but keep basic punctuation
        text = re.sub(r'[^\w\s.,!?;:()\[\]{}"\'-]', '', text)
        
        return text.strip()
    
    def extract_entities(self, text):
        doc = self.nlp(text)
        entities = []
        
        for ent in doc.ents:
            entities.append({
                'text': ent.text,
                'label': ent.label_,
                'start': ent.start_char,
                'end': ent.end_char
            })
        
        return entities
    
    def tokenize_and_process(self, text):
        # Tokenize
        tokens = word_tokenize(text.lower())
        
        # Remove stopwords and short words
        tokens = [token for token in tokens 
                 if token not in self.stop_words and len(token) > 2]
        
        # Stem tokens
        tokens = [self.stemmer.stem(token) for token in tokens]
        
        return tokens
    
    def calculate_features(self, document, tokens):
        return {
            'word_count': len(tokens),
            'title_length': len(document['title']),
            'has_description': bool(document['description']),
            'unique_words': len(set(tokens)),
            'avg_word_length': sum(len(token) for token in tokens) / len(tokens) if tokens else 0
        }
```

#### 3. Indexer
```python
import json
from collections import defaultdict
import math

class SearchIndexer:
    def __init__(self):
        self.inverted_index = defaultdict(lambda: defaultdict(list))
        self.document_frequencies = defaultdict(int)
        self.documents = {}
        self.doc_count = 0
    
    def index_document(self, processed_doc):
        doc_id = self.generate_doc_id(processed_doc['url'])
        
        # Store document
        self.documents[doc_id] = processed_doc
        self.doc_count += 1
        
        # Build inverted index
        token_positions = {}
        for position, token in enumerate(processed_doc['tokens']):
            if token not in token_positions:
                token_positions[token] = []
            token_positions[token].append(position)
        
        # Add to inverted index
        for token, positions in token_positions.items():
            self.inverted_index[token][doc_id] = {
                'tf': len(positions),
                'positions': positions
            }
            
            # Update document frequency
            if doc_id not in [doc['doc_id'] for doc in self.inverted_index[token].values()]:
                self.document_frequencies[token] += 1
    
    def calculate_tf_idf(self, token, doc_id):
        if token not in self.inverted_index or doc_id not in self.inverted_index[token]:
            return 0
        
        tf = self.inverted_index[token][doc_id]['tf']
        df = self.document_frequencies[token]
        
        # TF-IDF calculation
        tf_score = 1 + math.log(tf) if tf > 0 else 0
        idf_score = math.log(self.doc_count / df) if df > 0 else 0
        
        return tf_score * idf_score
    
    def build_posting_list(self, token):
        if token not in self.inverted_index:
            return []
        
        posting_list = []
        for doc_id, data in self.inverted_index[token].items():
            tf_idf = self.calculate_tf_idf(token, doc_id)
            posting_list.append({
                'doc_id': doc_id,
                'tf': data['tf'],
                'tf_idf': tf_idf,
                'positions': data['positions']
            })
        
        # Sort by TF-IDF score
        posting_list.sort(key=lambda x: x['tf_idf'], reverse=True)
        return posting_list
    
    def serialize_index(self):
        # Serialize index for storage
        serialized = {}
        for token, docs in self.inverted_index.items():
            serialized[token] = {
                'df': self.document_frequencies[token],
                'postings': self.build_posting_list(token)
            }
        
        return serialized
```

#### 4. Query Processor
```python
class QueryProcessor:
    def __init__(self, indexer):
        self.indexer = indexer
        self.processor = DocumentProcessor()
    
    def process_query(self, query_text):
        # Parse query
        parsed_query = self.parse_query(query_text)
        
        # Process query terms
        processed_terms = self.processor.tokenize_and_process(query_text)
        
        return {
            'original_query': query_text,
            'parsed_query': parsed_query,
            'terms': processed_terms
        }
    
    def parse_query(self, query_text):
        # Handle different query types
        if '"' in query_text:
            # Phrase query
            return self.parse_phrase_query(query_text)
        elif 'AND' in query_text or 'OR' in query_text:
            # Boolean query
            return self.parse_boolean_query(query_text)
        else:
            # Simple term query
            return {
                'type': 'term',
                'terms': query_text.split()
            }
    
    def parse_phrase_query(self, query_text):
        phrases = re.findall(r'"([^"]*)"', query_text)
        terms = re.sub(r'"[^"]*"', '', query_text).split()
        
        return {
            'type': 'phrase',
            'phrases': phrases,
            'terms': terms
        }
    
    def parse_boolean_query(self, query_text):
        # Simple boolean query parsing
        parts = re.split(r'\s+(AND|OR)\s+', query_text)
        
        return {
            'type': 'boolean',
            'expression': parts
        }
```

#### 5. Ranking Engine
```python
class RankingEngine:
    def __init__(self, indexer):
        self.indexer = indexer
        self.page_rank_scores = {}  # Would be loaded from PageRank computation
    
    def rank_documents(self, query, candidate_docs):
        scored_docs = []
        
        for doc_id in candidate_docs:
            score = self.calculate_relevance_score(query, doc_id)
            scored_docs.append({
                'doc_id': doc_id,
                'score': score,
                'document': self.indexer.documents[doc_id]
            })
        
        # Sort by score
        scored_docs.sort(key=lambda x: x['score'], reverse=True)
        
        return scored_docs
    
    def calculate_relevance_score(self, query, doc_id):
        # Combine multiple ranking factors
        tf_idf_score = self.calculate_tf_idf_score(query, doc_id)
        page_rank_score = self.page_rank_scores.get(doc_id, 0)
        freshness_score = self.calculate_freshness_score(doc_id)
        title_score = self.calculate_title_score(query, doc_id)
        
        # Weighted combination
        total_score = (
            0.4 * tf_idf_score +
            0.3 * page_rank_score +
            0.2 * freshness_score +
            0.1 * title_score
        )
        
        return total_score
    
    def calculate_tf_idf_score(self, query, doc_id):
        total_score = 0
        for term in query['terms']:
            total_score += self.indexer.calculate_tf_idf(term, doc_id)
        
        return total_score / len(query['terms']) if query['terms'] else 0
    
    def calculate_freshness_score(self, doc_id):
        document = self.indexer.documents[doc_id]
        timestamp = datetime.fromisoformat(document['timestamp'])
        age_days = (datetime.now() - timestamp).days
        
        # Fresh content gets higher score
        return 1 / (1 + age_days / 30)  # Decay over 30 days
    
    def calculate_title_score(self, query, doc_id):
        document = self.indexer.documents[doc_id]
        title_lower = document['title'].lower()
        
        matches = 0
        for term in query['terms']:
            if term in title_lower:
                matches += 1
        
        return matches / len(query['terms']) if query['terms'] else 0
```

#### 6. Search Service
```python
class SearchService:
    def __init__(self):
        self.indexer = SearchIndexer()
        self.query_processor = QueryProcessor(self.indexer)
        self.ranking_engine = RankingEngine(self.indexer)
        self.cache = {}
    
    def search(self, query_text, page=1, page_size=10):
        # Check cache first
        cache_key = f"{query_text}:{page}:{page_size}"
        if cache_key in self.cache:
            return self.cache[cache_key]
        
        # Process query
        processed_query = self.query_processor.process_query(query_text)
        
        # Find candidate documents
        candidate_docs = self.find_candidate_documents(processed_query)
        
        # Rank documents
        ranked_docs = self.ranking_engine.rank_documents(processed_query, candidate_docs)
        
        # Paginate results
        start = (page - 1) * page_size
        end = start + page_size
        results = ranked_docs[start:end]
        
        # Format results
        formatted_results = self.format_results(results, query_text)
        
        # Cache results
        self.cache[cache_key] = formatted_results
        
        return formatted_results
    
    def find_candidate_documents(self, query):
        candidate_docs = set()
        
        if query['parsed_query']['type'] == 'phrase':
            # Handle phrase queries
            candidate_docs = self.find_phrase_matches(query['parsed_query']['phrases'])
        elif query['parsed_query']['type'] == 'boolean':
            # Handle boolean queries
            candidate_docs = self.find_boolean_matches(query['parsed_query']['expression'])
        else:
            # Handle term queries
            for term in query['terms']:
                if term in self.indexer.inverted_index:
                    doc_ids = set(self.indexer.inverted_index[term].keys())
                    candidate_docs.update(doc_ids)
        
        return list(candidate_docs)
    
    def find_phrase_matches(self, phrases):
        # Implement phrase matching logic
        matches = set()
        
        for phrase in phrases:
            phrase_terms = phrase.split()
            phrase_matches = self.find_phrase_in_documents(phrase_terms)
            matches.update(phrase_matches)
        
        return matches
    
    def find_phrase_in_documents(self, phrase_terms):
        if not phrase_terms:
            return set()
        
        # Start with documents containing first term
        first_term = phrase_terms[0]
        if first_term not in self.indexer.inverted_index:
            return set()
        
        candidate_docs = set(self.indexer.inverted_index[first_term].keys())
        
        # Filter documents that contain the complete phrase
        phrase_matches = set()
        
        for doc_id in candidate_docs:
            if self.document_contains_phrase(doc_id, phrase_terms):
                phrase_matches.add(doc_id)
        
        return phrase_matches
    
    def document_contains_phrase(self, doc_id, phrase_terms):
        # Check if document contains all terms in sequence
        all_positions = {}
        
        for term in phrase_terms:
            if term not in self.indexer.inverted_index or doc_id not in self.indexer.inverted_index[term]:
                return False
            all_positions[term] = self.indexer.inverted_index[term][doc_id]['positions']
        
        # Check for sequential positions
        first_term_positions = all_positions[phrase_terms[0]]
        
        for start_pos in first_term_positions:
            found_phrase = True
            for i, term in enumerate(phrase_terms[1:], 1):
                expected_pos = start_pos + i
                if expected_pos not in all_positions[term]:
                    found_phrase = False
                    break
            
            if found_phrase:
                return True
        
        return False
    
    def format_results(self, results, query_text):
        formatted = []
        
        for result in results:
            doc = result['document']
            formatted.append({
                'url': doc['url'],
                'title': doc['title'],
                'snippet': self.generate_snippet(doc['content'], query_text),
                'score': result['score']
            })
        
        return formatted
    
    def generate_snippet(self, content, query_text):
        # Generate snippet with query terms highlighted
        query_terms = query_text.lower().split()
        content_lower = content.lower()
        
        # Find best position for snippet
        best_position = 0
        max_matches = 0
        
        for i in range(0, len(content) - 160, 20):
            snippet = content[i:i+160].lower()
            matches = sum(1 for term in query_terms if term in snippet)
            
            if matches > max_matches:
                max_matches = matches
                best_position = i
        
        # Generate snippet
        snippet = content[best_position:best_position+160]
        
        # Highlight query terms
        for term in query_terms:
            snippet = re.sub(
                f'({re.escape(term)})',
                r'<strong>\1</strong>',
                snippet,
                flags=re.IGNORECASE
            )
        
        return snippet + "..."
```

## Auto-complete Service

```python
class AutoCompleteService:
    def __init__(self):
        self.trie = TrieNode()
        self.popular_queries = {}
    
    def build_suggestions(self, query_logs):
        # Build from query logs
        for query, frequency in query_logs.items():
            self.add_query(query, frequency)
    
    def add_query(self, query, frequency):
        # Add to trie
        node = self.trie
        for char in query.lower():
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        
        node.is_end = True
        node.frequency = frequency
    
    def get_suggestions(self, prefix, max_suggestions=10):
        # Find prefix node
        node = self.trie
        for char in prefix.lower():
            if char not in node.children:
                return []
            node = node.children[char]
        
        # Collect suggestions
        suggestions = []
        self.collect_suggestions(node, prefix, suggestions)
        
        # Sort by frequency
        suggestions.sort(key=lambda x: x['frequency'], reverse=True)
        
        return suggestions[:max_suggestions]
    
    def collect_suggestions(self, node, prefix, suggestions):
        if node.is_end:
            suggestions.append({
                'query': prefix,
                'frequency': node.frequency
            })
        
        for char, child_node in node.children.items():
            self.collect_suggestions(child_node, prefix + char, suggestions)

class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.frequency = 0
```

## Performance Optimizations

### 1. Distributed Indexing
```python
class DistributedIndexer:
    def __init__(self, num_shards=10):
        self.num_shards = num_shards
        self.shards = [SearchIndexer() for _ in range(num_shards)]
    
    def get_shard(self, term):
        return hash(term) % self.num_shards
    
    def index_document(self, processed_doc):
        # Distribute terms across shards
        for token in processed_doc['tokens']:
            shard_id = self.get_shard(token)
            self.shards[shard_id].index_document(processed_doc)
    
    def search_across_shards(self, query):
        # Search all relevant shards
        results = []
        for shard in self.shards:
            shard_results = shard.search(query)
            results.extend(shard_results)
        
        return results
```

### 2. Caching Strategy
```python
class SearchCache:
    def __init__(self):
        self.query_cache = {}
        self.result_cache = {}
        self.cache_stats = {'hits': 0, 'misses': 0}
    
    def get_cached_results(self, query_key):
        if query_key in self.query_cache:
            self.cache_stats['hits'] += 1
            return self.query_cache[query_key]
        
        self.cache_stats['misses'] += 1
        return None
    
    def cache_results(self, query_key, results):
        # LRU cache implementation
        if len(self.query_cache) > 10000:
            # Remove oldest entries
            oldest_key = next(iter(self.query_cache))
            del self.query_cache[oldest_key]
        
        self.query_cache[query_key] = results
```

### 3. Real-time Updates
```python
class RealTimeIndexer:
    def __init__(self, main_indexer):
        self.main_indexer = main_indexer
        self.delta_index = SearchIndexer()
        self.pending_updates = []
    
    def add_document(self, document):
        # Add to delta index for immediate search
        self.delta_index.index_document(document)
        
        # Queue for batch processing
        self.pending_updates.append(document)
        
        # Trigger batch update if needed
        if len(self.pending_updates) >= 1000:
            self.merge_updates()
    
    def merge_updates(self):
        # Merge delta index with main index
        for document in self.pending_updates:
            self.main_indexer.index_document(document)
        
        # Clear delta index and pending updates
        self.delta_index = SearchIndexer()
        self.pending_updates = []
    
    def search(self, query):
        # Search both main and delta indexes
        main_results = self.main_indexer.search(query)
        delta_results = self.delta_index.search(query)
        
        # Merge and deduplicate results
        combined_results = main_results + delta_results
        return self.deduplicate_results(combined_results)
```

## Monitoring & Analytics

### Search Analytics
```python
class SearchAnalytics:
    def __init__(self):
        self.query_logs = []
        self.metrics = {
            'total_queries': 0,
            'avg_response_time': 0,
            'cache_hit_rate': 0,
            'zero_results_rate': 0
        }
    
    def log_query(self, query, results_count, response_time):
        self.query_logs.append({
            'query': query,
            'results_count': results_count,
            'response_time': response_time,
            'timestamp': datetime.now()
        })
        
        self.update_metrics()
    
    def update_metrics(self):
        if not self.query_logs:
            return
        
        recent_logs = self.query_logs[-1000:]  # Last 1000 queries
        
        self.metrics['total_queries'] = len(self.query_logs)
        self.metrics['avg_response_time'] = sum(log['response_time'] for log in recent_logs) / len(recent_logs)
        self.metrics['zero_results_rate'] = sum(1 for log in recent_logs if log['results_count'] == 0) / len(recent_logs)
    
    def get_popular_queries(self, limit=10):
        query_counts = {}
        for log in self.query_logs:
            query = log['query']
            query_counts[query] = query_counts.get(query, 0) + 1
        
        return sorted(query_counts.items(), key=lambda x: x[1], reverse=True)[:limit]
```

## Scalability Considerations

### Database Schema (PostgreSQL)
```sql
-- Documents table
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    url VARCHAR(2048) UNIQUE NOT NULL,
    title TEXT,
    content TEXT,
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_crawled TIMESTAMP
);

-- Index table
CREATE TABLE search_index (
    id SERIAL PRIMARY KEY,
    term VARCHAR(255) NOT NULL,
    document_id INTEGER REFERENCES documents(id),
    tf_idf FLOAT,
    positions INTEGER[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for performance
CREATE INDEX idx_search_term ON search_index(term);
CREATE INDEX idx_document_url ON documents(url);
CREATE INDEX idx_tf_idf ON search_index(tf_idf DESC);
```

### Deployment Architecture
```yaml
# docker-compose.yml
version: '3.8'
services:
  crawler:
    image: search-engine/crawler
    replicas: 3
    environment:
      - KAFKA_BROKERS=kafka:9092
      - REDIS_URL=redis:6379
  
  indexer:
    image: search-engine/indexer
    replicas: 5
    environment:
      - ELASTICSEARCH_URL=elasticsearch:9200
      - POSTGRES_URL=postgres:5432
  
  search-api:
    image: search-engine/api
    replicas: 10
    ports:
      - "8080:8080"
    environment:
      - REDIS_URL=redis:6379
      - ELASTICSEARCH_URL=elasticsearch:9200
  
  elasticsearch:
    image: elasticsearch:7.10
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
  
  redis:
    image: redis:alpine
    
  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=search_engine
      - POSTGRES_USER=search
      - POSTGRES_PASSWORD=password
```

Search Engine System này cung cấp foundation đầy đủ cho một search engine có khả năng mở rộng cao với các tính năng như crawling, indexing, ranking, và real-time search. 