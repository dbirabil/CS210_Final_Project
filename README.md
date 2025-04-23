# CS210_Final_Project
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <list>
#include <tuple>

using namespace std;

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
           return true;
       }
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
           }
       }
   }

   void printCache() {
       cout << "Cache contents (most recent first):" << endl;
       for (const auto& entry : cacheList) {
           cout << "City: " << get<0>(entry) << ", Country: " << get<1>(entry) << ", Population: " << get<2>(entry) << endl;
       }
   }
};

bool searchInCSV(const string& fileName, const string& city, const string& countryCode, int& population) {
    ifstream file(fileName);
    if (!file.is_open()) {
        cerr << "Error: Could not open file: " << fileName << endl;
        return false;
    }

    string line;
    while (getline(file, line)) {
        stringstream ss(line);
        string csvCity, csvCountryCode, csvPopulation;

        getline(ss, csvCountryCode, ',');
        getline(ss, csvCity, ',');
        getline(ss, csvPopulation, ',');

        if (csvCity == city && csvCountryCode == countryCode) {
            try {
                population = static_cast<int>(stof(csvPopulation));
            }
            catch (const invalid_argument& e) {
                cerr << "Error: Invalid population value in CSV for " << city << ", " << countryCode << endl;
                file.close();
                return false;
            }
            catch (const out_of_range& e) {
                cerr << "Error: Population value out of range in CSV for " << city << ", " << countryCode << endl;
                file.close();
                return false;
            }
            file.close();
            return true;
        }
    }

    cerr << "Error: City and country code not found in file." << endl;
    file.close();
    return false;
}


int main() {
   const string fileName = "world_cities.csv";
   LRUCache cache(10);

   while (true) {
       string city, countryCode;
       cout << "Enter city name (or 'exit' to quit): ";
       getline(cin, city);
       if (city == "exit") break;

       cout << "Enter country code: ";
       getline(cin, countryCode);

       int population;
       if (cache.getEntry(city, countryCode, population)) { 
           cout << "Population (from cache): " << population << endl;
       }
       else if (searchInCSV(fileName, city, countryCode, population)) {
           cout << "Population (from file): " << population << endl;
           cache.put(city, countryCode, population);
       }
       else {
           cout << "City and country code not found." << endl;
       }

       cache.printCache();
   }

   return 0;
}
