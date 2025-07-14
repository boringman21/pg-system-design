# Search Systems Architecture

**Tags**: #search #elasticsearch #system-components #information-retrieval
**Date**: 2024-01-01

## 🎯 Tổng quan

Search Systems cho phép users tìm kiếm và retrieve thông tin relevant từ large datasets một cách nhanh chóng và chính xác.

## 🏗️ Core Components

### **Search Index**
```
- Inverted Index: Maps terms to documents
- Forward Index: Maps documents to terms  
- Metadata Index: Document properties
- Scoring Index: Relevance calculations
```

### **Query Processing Pipeline**
```
1. Query Parsing & Analysis
2. Query Expansion & Synonyms
3. Index Lookup
4. Scoring & Ranking
5. Result Aggregation
6. Response Formatting
```

### **Key Technologies**
```
Elasticsearch:
✅ Distributed search engine
✅ RESTful API
✅ Real-time indexing
✅ Aggregations support

Solr:
✅ Enterprise search platform
✅ Rich query syntax
✅ Faceted search
✅ Clustering support

Custom Solutions:
✅ Specialized algorithms
✅ Domain-specific optimization
✅ Performance tuning
✅ Cost control
```

## 🔍 Search Algorithms

### **Scoring Models**
```python
# TF-IDF Scoring
def tf_idf_score(term, document, corpus):
    tf = term_frequency(term, document)
    idf = inverse_document_frequency(term, corpus)
    return tf * idf

# BM25 Scoring (Better than TF-IDF)
def bm25_score(term, document, corpus, k1=1.5, b=0.75):
    tf = term_frequency(term, document)
    idf = inverse_document_frequency(term, corpus)
    doc_len = len(document)
    avg_doc_len = average_document_length(corpus)
    
    score = idf * (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * (doc_len / avg_doc_len)))
    return score

# Modern ML-based scoring
class NeuralSearchRanker:
    def __init__(self):
        self.bert_model = load_pretrained_bert()
        self.embedding_cache = {}
    
    def score_query_document(self, query: str, document: str):
        query_embedding = self.get_embedding(query)
        doc_embedding = self.get_embedding(document)
        
        # Cosine similarity
        similarity = cosine_similarity(query_embedding, doc_embedding)
        return similarity
```

### **Query Understanding**
```python
class QueryProcessor:
    def __init__(self):
        self.analyzer = TextAnalyzer()
        self.spell_checker = SpellChecker()
        self.synonym_expander = SynonymExpander()
    
    def process_query(self, raw_query: str):
        # 1. Spell correction
        corrected = self.spell_checker.correct(raw_query)
        
        # 2. Tokenization and normalization
        tokens = self.analyzer.tokenize(corrected)
        
        # 3. Synonym expansion
        expanded_tokens = self.synonym_expander.expand(tokens)
        
        # 4. Query intent detection
        intent = self.detect_intent(expanded_tokens)
        
        return {
            'original': raw_query,
            'corrected': corrected,
            'tokens': tokens,
            'expanded_tokens': expanded_tokens,
            'intent': intent
        }
```

## 📊 Performance Optimization

### **Indexing Strategies**
```
Sharding:
- Horizontal partitioning by document ID
- Geographic sharding
- Time-based sharding

Replication:
- Read replicas for query load
- Hot-standby for failover
- Cross-region replication

Optimization:
- Index warming
- Bulk indexing
- Refresh intervals
- Segment merging
```

### **Caching Layers**
```python
class SearchCacheManager:
    def __init__(self):
        self.query_cache = Redis()  # Query result cache
        self.filter_cache = Redis()  # Filter result cache
        self.aggregation_cache = Redis()  # Aggregation cache
    
    async def get_cached_results(self, query_hash: str):
        cached = await self.query_cache.get(query_hash)
        if cached:
            return json.loads(cached)
        return None
    
    async def cache_results(self, query_hash: str, results: dict, ttl: int = 300):
        await self.query_cache.setex(
            query_hash, 
            ttl, 
            json.dumps(results)
        )
```

## 💡 Best Practices

### **Relevance Tuning**
```
- A/B test ranking algorithms
- Monitor click-through rates
- Implement feedback loops
- Use machine learning for personalization
- Analyze query logs for patterns
```

### **Scalability Patterns**
```
- Separate read/write clusters
- Use appropriate shard sizes
- Implement circuit breakers
- Monitor resource usage
- Plan capacity proactively
```

---

**Key Metrics:**
- Query latency (p95, p99)
- Index size and growth
- Cache hit ratios
- Relevance scores
- User engagement metrics

**Remember**: Search quality is as important as search speed! 🔍 