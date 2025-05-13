# CS210_Final_Project
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <list>
#include <tuple>


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


class LRUCache {
private:
    size_t capacity;
    list<tuple<string, string, int>> cacheList;
    unordered_map<string, list<tuple<string, string, int>>::iterator> cacheMap;

    string generateKey(const string& city, const string& countryCode) {
        return city + "," + countryCode;
    }

public:
    LRUCache(size_t cap) : capacity(cap) {}

    bool getEntry(const string& city, const string& countryCode, int& population) {
        string key = generateKey(city, countryCode);
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            cacheList.splice(cacheList.begin(), cacheList, it->second);
            population = get<2>(*(it->second));
            cout << "[Cache Hit]" << endl;
            return true;
        }
        cout << "[Cache Miss]" << endl;
        return false;
    }

    void put(const string& city, const string& countryCode, int population) {
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
                cout << "[Cache Eviction]" << endl;
            }
        }
    }

    void printCache() {
        cout << "\nCache contents (most recent first):" << endl;
        for (const auto& entry : cacheList) {
            cout << "City: " << get<0>(entry)
                << ", Country: " << get<1>(entry)
                << ", Population: " << get<2>(entry) << endl;
        }
        cout << endl;
    }
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
    int count = 0;
    while (getline(file, line)) {
        stringstream ss(line);
        string countryCode, city, populationStr;
        getline(ss, countryCode, ',');
        getline(ss, city, ',');
        getline(ss, populationStr, ',');

        try {
            int population = stoi(populationStr);
            trie.insert(trim(city), trim(countryCode), population);
            count++;
        }
        catch (...) {
            cerr << "Skipping invalid row: " << line << endl;
        }
    }

    file.close();
    cout << "Loaded " << count << " cities into Trie." << endl;
}



int main() {
    const string fileName = "world_cities.csv";
    CityTrie trie;
    loadCSVIntoTrie(fileName, trie);

    LRUCache cache(10);

    while (true) {
        string city, countryCode;
        cout << "Enter city name (or 'exit' to quit): ";
        getline(cin, city);
        if (city == "exit") break;

        cout << "Enter country code: ";
        getline(cin, countryCode);

        city = trim(city);
        countryCode = trim(countryCode);

        int population;
        if (cache.getEntry(city, countryCode, population)) {
            cout << "Population (from cache): " << population << endl;
        }
        else if (trie.search(city, countryCode, population)) {
            cout << "Population (from Trie): " << population << endl;
            cache.put(city, countryCode, population);
        }
        else {
            cout << "City and country code not found." << endl;
        }

        cache.printCache();
    }

    return 0;
}







