# CS210_Final_Project
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <list>
#include <tuple>
#include <vector>
#include <random>
#include <chrono>
#include <iomanip>
#include <algorithm>

using namespace std;



struct TrieNode {
    unordered_map<char, TrieNode*> children;
    unordered_map<string, int> countryPopulationMap;
    bool isEndOfCity = false;
};

class CityTrie {
private:
    TrieNode* root;

    char normalize(char c) {
        return tolower(c);
    }

public:
    CityTrie() {
        root = new TrieNode();
    }

    void insert(const string& city, const string& countryCode, int population) {
        TrieNode* node = root;
        for (char c : city) {
            c = normalize(c);
            if (!node->children[c]) {
                node->children[c] = new TrieNode();
            }
            node = node->children[c];
        }
        node->isEndOfCity = true;
        node->countryPopulationMap[countryCode] = population;
    }

    bool search(const string& city, const string& countryCode, int& population) {
        TrieNode* node = root;
        for (char c : city) {
            c = normalize(c);
            if (!node->children.count(c)) return false;
            node = node->children[c];
        }
        if (node->isEndOfCity && node->countryPopulationMap.count(countryCode)) {
            population = node->countryPopulationMap[countryCode];
            return true;
        }
        return false;
    }
};



class CacheBase {
public:
    virtual bool getEntry(const string& city, const string& countryCode, int& population) = 0;
    virtual void put(const string& city, const string& countryCode, int population) = 0;
    virtual void resetStats() = 0;
    virtual int getHits() const = 0;
    virtual int getMisses() const = 0;
    virtual ~CacheBase() {}
};


class LRUCache : public CacheBase {
private:
    size_t capacity;
    list<tuple<string, string, int>> cacheList;
    unordered_map<string, list<tuple<string, string, int>>::iterator> cacheMap;
    int hits = 0, misses = 0;

    string generateKey(const string& city, const string& countryCode) {
        return city + "," + countryCode;
    }

public:
    LRUCache(size_t cap) : capacity(cap) {}

    bool getEntry(const string& city, const string& countryCode, int& population) override {
        string key = generateKey(city, countryCode);
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            cacheList.splice(cacheList.begin(), cacheList, it->second);
            population = get<2>(*(it->second));
            hits++;
            return true;
        }
        misses++;
        return false;
    }

    void put(const string& city, const string& countryCode, int population) override {
        string key = generateKey(city, countryCode);
        auto it = cacheMap.find(key);

        if (it != cacheMap.end()) {
            cacheList.splice(cacheList.begin(), cacheList, it->second);
            *(it->second) = make_tuple(city, countryCode, population);
        }
        else {
            cacheList.emplace_front(city, countryCode, population);
            cacheMap[key] = cacheList.begin();
            if (cacheList.size() > capacity) {
                auto last = cacheList.back();
                cacheMap.erase(generateKey(get<0>(last), get<1>(last)));
                cacheList.pop_back();
            }
        }
    }

    void resetStats() override { hits = misses = 0; }
    int getHits() const override { return hits; }
    int getMisses() const override { return misses; }
};


class FIFOCache : public CacheBase {
private:
    size_t capacity;
    list<tuple<string, string, int>> cacheList;
    unordered_map<string, list<tuple<string, string, int>>::iterator> cacheMap;
    int hits = 0, misses = 0;

    string generateKey(const string& city, const string& countryCode) {
        return city + "," + countryCode;
    }

public:
    FIFOCache(size_t cap) : capacity(cap) {}

    bool getEntry(const string& city, const string& countryCode, int& population) override {
        string key = generateKey(city, countryCode);
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            population = get<2>(*(it->second));
            hits++;
            return true;
        }
        misses++;
        return false;
    }

    void put(const string& city, const string& countryCode, int population) override {
        string key = generateKey(city, countryCode);
        if (cacheMap.find(key) == cacheMap.end()) {
            cacheList.emplace_back(city, countryCode, population);
            cacheMap[key] = prev(cacheList.end());
            if (cacheList.size() > capacity) {
                auto first = cacheList.front();
                cacheMap.erase(generateKey(get<0>(first), get<1>(first)));
                cacheList.pop_front();
            }
        }
    }

    void resetStats() override { hits = misses = 0; }
    int getHits() const override { return hits; }
    int getMisses() const override { return misses; }
};



class RandomCache : public CacheBase {
private:
    size_t capacity;
    vector<tuple<string, string, int>> cacheVec;
    unordered_map<string, int> cacheMap;
    int hits = 0, misses = 0;

    string generateKey(const string& city, const string& countryCode) {
        return city + "," + countryCode;
    }

public:
    RandomCache(size_t cap) : capacity(cap) {}

    bool getEntry(const string& city, const string& countryCode, int& population) override {
        string key = generateKey(city, countryCode);
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            population = get<2>(cacheVec[it->second]);
            hits++;
            return true;
        }
        misses++;
        return false;
    }

    void put(const string& city, const string& countryCode, int population) override {
        string key = generateKey(city, countryCode);
        if (cacheMap.count(key)) return;

        if (cacheVec.size() >= capacity) {
            int idx = rand() % cacheVec.size();
            cacheMap.erase(generateKey(get<0>(cacheVec[idx]), get<1>(cacheVec[idx])));
            cacheVec[idx] = make_tuple(city, countryCode, population);
            cacheMap[key] = idx;
        }
        else {
            cacheVec.push_back(make_tuple(city, countryCode, population));
            cacheMap[key] = cacheVec.size() - 1;
        }
    }

    void resetStats() override { hits = misses = 0; }
    int getHits() const override { return hits; }
    int getMisses() const override { return misses; }
};



string trim(const string& str) {
    size_t first = str.find_first_not_of(" \t\r\n");
    size_t last = str.find_last_not_of(" \t\r\n");
    return (first == string::npos) ? "" : str.substr(first, last - first + 1);
}

void loadCSVIntoTrie(const string& fileName, CityTrie& trie) {
    ifstream file(fileName);
    if (!file.is_open()) {
        cerr << "Error opening file." << endl;
        return;
    }

    string line;
    while (getline(file, line)) {
        stringstream ss(line);
        string countryCode, city, populationStr;
        getline(ss, countryCode, ',');
        getline(ss, city, ',');
        getline(ss, populationStr, ',');

        try {
            int population = stoi(populationStr);
            trie.insert(trim(city), trim(countryCode), population);
        }
        catch (...) {
            continue;
        }
    }

    file.close();
}

void loadCityList(const string& fileName, vector<pair<string, string>>& cities) {
    ifstream file(fileName);
    string line;
    while (getline(file, line)) {
        stringstream ss(line);
        string countryCode, city, population;
        getline(ss, countryCode, ',');
        getline(ss, city, ',');
        getline(ss, population, ',');

        if (!city.empty() && !countryCode.empty())
            cities.emplace_back(trim(city), trim(countryCode));
    }
}

void runTest(CacheBase* cache, CityTrie& trie, const vector<pair<string, string>>& queries) {
   cache->resetStats();
   int totalLookups = queries.size();
   double totalTimeMs = 0.0;

   for (const auto& query : queries) {
       const string& city = query.first;
       const string& countryCode = query.second; 

       int population;
       auto start = chrono::high_resolution_clock::now();
       if (!cache->getEntry(city, countryCode, population)) {
           if (trie.search(city, countryCode, population)) {
               cache->put(city, countryCode, population);
           }
       }
       auto end = chrono::high_resolution_clock::now();
       totalTimeMs += chrono::duration<double, milli>(end - start).count();
   }

   double avgTime = totalTimeMs / totalLookups;
   double hitRatio = 100.0 * cache->getHits() / totalLookups;

   cout << fixed << setprecision(3);
   cout << "Total lookups: " << totalLookups << endl;
   cout << "Cache Hits: " << cache->getHits()
       << ", Misses: " << cache->getMisses() << endl;
   cout << "Average Lookup Time: " << avgTime << " ms" << endl;
   cout << "Cache Hit Ratio: " << hitRatio << "%" << endl;
}



int main() {
    const string fileName = "world_cities.csv";
    CityTrie trie;
    loadCSVIntoTrie(fileName, trie);

    vector<pair<string, string>> allCities;
    loadCityList(fileName, allCities);

    vector<pair<string, string>> testQueries;
    for (int i = 0; i < 1000; ++i) {
        if (i % 3 == 0 && !testQueries.empty()) {
            testQueries.push_back(testQueries[rand() % testQueries.size()]);
        }
        else {
            testQueries.push_back(allCities[rand() % allCities.size()]);
        }
    }

    cout << "\n LRU Cache\n";
    LRUCache lru(50);
    runTest(&lru, trie, testQueries);

    cout << "\n FIFO Cache \n";
    FIFOCache fifo(50);
    runTest(&fifo, trie, testQueries);

    cout << "\nRand Replacement Cache\n";
    RandomCache rnd(50);
    runTest(&rnd, trie, testQueries);

    return 0;
}






