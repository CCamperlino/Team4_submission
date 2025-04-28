#include <iostream>
#include <vector>
#include <string>
#include <fstream>
#include <algorithm>
#include <cstdlib>
#include <ctime>
#include <map>

using namespace std;

// ====== STRUCTS ======
struct Item {
    string id;
    string name;
    int quantity;
    int reorderPoint;
    string locationId;
};

struct Location {
    string id;
    string description;
};

struct DamagedItem {
    string itemId;
    int quantity;
};

struct Order {
    int customersownId;
    int orderId;
    string itemName;
    int amount;
    string status;
};

struct itemBin {
    string name;
    int count;
};

struct ShippingLog {
    string itemName;
    int quantity;
    string type; // "Shipped" or "Received"
    string timestamp;
};

// ====== GLOBAL VECTORS ======
vector<Item> inventory;
vector<Location> locations;
vector<DamagedItem> damagedItems;
vector<ShippingLog> shippingLogs;
itemBin bins[10];

// ====== UTILITY ======
string getLineInput(const string& prompt) {
    string input;
    cout << prompt;
    getline(cin, input);
    return input;
}

int generateIDs(int min, int max) {
    return rand() % (max - min + 1) + min;
}

string currentTimestamp() {
    time_t now = time(0);
    char buf[80];
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", localtime(&now));
    return string(buf);
}

// ====== ORDER FUNCTIONS ======
Order makeOrder(const string& itemName, int quantity, const string& status) {
    Order order;
    order.customersownId = generateIDs(100, 999);
    order.orderId = generateIDs(1000, 9999);
    order.itemName = itemName;
    order.amount = quantity;
    order.status = status;
    return order;
}

void detailedOrder(const Order& order) {
    cout << "\nYour Order:\\n";
    cout << "Customer #: " << order.customersownId << "\nOrder #: " << order.orderId;
    cout << "\nProduct: " << order.itemName << "\nQuantity: " << order.amount;
    cout << "\nStatus: " << order.status << endl;
}

// ====== STOCK TRACKING ======
void syncBinsWithInventory() {
    int i = 0;
    for (const auto& item : inventory) {
        if (i < 5) {
            bins[i].name = item.name;
            bins[i].count = item.quantity;
            ++i;
        }
    }
}

// ====== LOCATION MANAGEMENT ======
void addLocation() {
    cin.ignore();
    Location loc;
    loc.id = getLineInput("Enter Location ID: ");
    loc.description = getLineInput("Enter Description: ");
    locations.push_back(loc);
    cout << "Location added.\n";
}

void listLocations() {
    cout << "\n--- Defined Locations ---\n";
    if (locations.empty()) {
        cout << "No locations available.\n";
        return;
    }
    for (const auto& loc : locations) {
        cout << "ID: " << loc.id << " | Description: " << loc.description << endl;
    }
}

// ====== INVENTORY MANAGEMENT ======
void addItem() {
    if (locations.empty()) {
        cout << "No locations defined. Please add a location first.\n";
        return;
    }

    cin.ignore();
    Item newItem;
    newItem.id = getLineInput("Enter Item ID: ");
    newItem.name = getLineInput("Enter Item Name: ");

    cout << "Enter Quantity: ";
    while (!(cin >> newItem.quantity) || newItem.quantity < 0) {
        cout << "Invalid quantity. Enter again: ";
        cin.clear(); cin.ignore(1000, '\n');
    }

    cout << "Enter Reorder Point: ";
    while (!(cin >> newItem.reorderPoint) || newItem.reorderPoint < 0) {
        cout << "Invalid reorder point. Enter again: ";
        cin.clear(); cin.ignore(1000, '\n');
    }

    listLocations();
    cin.ignore();

    bool validLocation = false;
    do {
        newItem.locationId = getLineInput("Enter Location ID from list: ");
        for (const auto& loc : locations) {
            if (loc.id == newItem.locationId) {
                validLocation = true;
                break;
            }
        }
        if (!validLocation) cout << "Invalid Location ID. Try again.\n";
    } while (!validLocation);

    inventory.push_back(newItem);
    syncBinsWithInventory();
    cout << "Item added successfully.\n";
}

void listItems() {
    cout << "\n--- Inventory Items ---\n";
    if (inventory.empty()) {
        cout << "No items found.\n";
        return;
    }
    for (const auto& item : inventory) {
        cout << "ID: " << item.id << " | " << item.name << " | Qty: " << item.quantity;
        cout << " | Reorder: " << item.reorderPoint << " | Loc: " << item.locationId;
        if (item.quantity <= item.reorderPoint) cout << " (LOW STOCK!)";
        cout << endl;
    }
}

// ====== REORDER POINT CHECK ======
void checkReorderAlerts() {
    cout << "\n--- Reorder Point Alerts ---\n";
    bool found = false;
    for (const auto& item : inventory) {
        if (item.quantity <= item.reorderPoint) {
            cout << "ALERT: " << item.name << " (Qty: " << item.quantity << ")\n";
            found = true;
        }
    }
    if (!found) cout << "All stocks above reorder levels.\n";
}

// ====== SEARCH ITEMS ======
void searchInventory() {
    cout << "\nSearch Inventory (case sensitive, 'exit' to stop)\n";
    string input;
    cin.ignore();
    while (true) {
        cout << "Item name: ";
        getline(cin, input);
        if (input == "exit") break;
        bool found = false;
        for (const auto& item : inventory) {
            if (item.name.find(input) != string::npos) {
                cout << "Found: " << item.name << " (Qty: " << item.quantity << ")\n";
                found = true;
            }
        }
        if (!found) cout << "Item not found.\n";
    }
}

// ====== ORDER FULFILLMENT ======
void placeOrder() {
    cin.ignore();
    string name;
    int qty;
    cout << "Enter product: ";
    getline(cin, name);
    cout << "Enter quantity: ";
    cin >> qty;

    if (qty <= 0) {
        cout << "Invalid quantity.\n";
        return;
    }

    for (auto& item : inventory) {
        if (item.name == name) {
            if (item.quantity < qty) {
                cout << "Not enough stock!\n";
                return;
            }
            item.quantity -= qty;
            syncBinsWithInventory();

            Order newOrder = makeOrder(name, qty, "Fulfilled");
            detailedOrder(newOrder);

            ShippingLog log = {name, qty, "Shipped", currentTimestamp()};
            shippingLogs.push_back(log);
            return;
        }
    }
    cout << "Product not found.\n";
}

// ====== RECEIVE STOCK ======
void receiveStock() {
    cin.ignore();
    string name = getLineInput("Enter item to receive: ");
    int qty;
    cout << "Enter quantity received: ";
    cin >> qty;

    for (auto& item : inventory) {
        if (item.name == name) {
            item.quantity += qty;
            syncBinsWithInventory();
            cout << "Stock updated.\n";

            ShippingLog log = {name, qty, "Received", currentTimestamp()};
            shippingLogs.push_back(log);
            return;
        }
    }
    cout << "Item not found in inventory.\n";
}

// ====== DAMAGE FUNCTIONS ======
void reportDamagedItem() {
    cin.ignore();
    string name = getLineInput("Enter item name: ");
    int qty;
    cout << "Enter quantity damaged: ";
    cin >> qty;

    for (auto& item : inventory) {
        if (item.name == name) {
            if (item.quantity >= qty) {
                item.quantity -= qty;
                damagedItems.push_back({item.id, qty});
                syncBinsWithInventory();
                cout << "Damage reported.\n";
            } else cout << "Not enough stock.\n";
            return;
        }
    }
    cout << "Item not found.\n";
}

void showDamagedItems() {
    cout << "\nDamaged Items Log\n";
    if (damagedItems.empty()) {
        cout << "No damage reports.\n";
        return;
    }
    for (const auto& d : damagedItems) {
        cout << "Item ID: " << d.itemId << " | Quantity: " << d.quantity << endl;
    }
}

// ====== BINS ======
void displayBins() {
    cout << "\n--- Bins ---\n";
    for (int i = 0; i < 5; ++i) {
        cout << "Bin#" << i+1 << ": " << bins[i].name << " (" << bins[i].count << " items)\n";
    }
}

// ====== SHIPPING LOGS ======
void viewShippingLogs() {
    cout << "\n--- Shipping/Receiving Logs ---\n";
    if (shippingLogs.empty()) {
        cout << "No logs available.\n";
        return;
    }
    for (const auto& log : shippingLogs) {
        cout << "[" << log.timestamp << "] " << log.type << " " << log.quantity << " of " << log.itemName << endl;
    }
}

// ====== INVENTORY REPORT ======
void generateInventoryReport() {
    cout << "\n--- Inventory Report ---\n";
    cout << "Items: " << inventory.size() << "\nDamaged: " << damagedItems.size() << "\\n";
}

// ====== SAVE & LOAD ======
void saveData() {
    ofstream fout("inventory.txt");
    for (const auto& item : inventory) {
        fout << item.id << "," << item.name << "," << item.quantity << "," << item.reorderPoint << "," << item.locationId << "\n";
    }
    fout.close();

    fout.open("locations.txt");
    for (const auto& loc : locations) {
        fout << loc.id << "," << loc.description << "\n";
    }
    fout.close();

    fout.open("damaged.txt");
    for (const auto& dmg : damagedItems) {
        fout << dmg.itemId << "," << dmg.quantity << "\n";
    }
    fout.close();

    fout.open("shipping_logs.txt");
    for (const auto& log : shippingLogs) {
        fout << log.itemName << "," << log.quantity << "," << log.type << "," << log.timestamp << "\\n";
    }
    fout.close();

    cout << "Data saved successfully.\n";
}

void loadData() {
    // You can fill this similarly to saveData if needed later
    cout << "(Loading feature placeholder)\n";
}

// ====== MAIN MENU ======
int main() {
    srand(time(0));
    syncBinsWithInventory();

    int choice;
    do {
        cout << "\n==== Inventory Management ====\n";
        cout << "1. Add Item\n2. List Items\n3. Add Location\n4. List Locations\n5. Report/View Damage\n";
        cout << "6. Reorder Alerts\n7. Save Data\n8. Search Inventory\n9. Place Order\n10. View Bins\n";
        cout << "11. Generate Report\n12. Receive Stock\n13. View Shipping Logs\n0. Exit\nChoice: ";
        cin >> choice;

        switch (choice) {
            case 1: addItem(); break;
            case 2: listItems(); break;
            case 3: addLocation(); break;
            case 4: listLocations(); break;
            case 5: { int x; cout << "1. Report 2. View: "; cin >> x; if (x==1) reportDamagedItem(); else showDamagedItems(); } break;
            case 6: checkReorderAlerts(); break;
            case 7: saveData(); break;
            case 8: searchInventory(); break;
            case 9: placeOrder(); break;
            case 10: displayBins(); break;
            case 11: generateInventoryReport(); break;
            case 12: receiveStock(); break;
            case 13: viewShippingLogs(); break;
            case 0: cout << "Goodbye.\n"; break;
            default: cout << "Invalid option!\n"; break;
        }
    } while (choice != 0);

    return 0;
}