# Implementasi-Design-Pattern-PPL-A10

### pesanmakan.cpp
```
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <memory>
#include <limits>
#include <sstream>

using namespace std;

// ==========================================
// UTILITY: FUNGSI INPUT ANTI-HANG
// ==========================================
int getValidIntInput() {
    string input;
    int choice;
    while (true) {
        getline(cin, input);
        stringstream ss(input);
        if (ss >> choice) break;
        cout << "[Error] Input tidak valid. Masukkan angka: ";
    }
    return choice;
}

// ==========================================
// 1. STRATEGY PATTERN (UC02 Memilih Metode Pembayaran)
// ==========================================
class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual void pay(double amount) const = 0;
};

class CODPayment : public PaymentStrategy { // UC02.1
public:
    void pay(double amount) const override {
        cout << "[Sistem] Metode COD dipilih. Siapkan Rp" << amount << " saat pesanan tiba.\n";
    }
};

class TransferPayment : public PaymentStrategy { // UC02.2
private:
    string bankAccount;
public:
    TransferPayment(const string& account) : bankAccount(account) {}
    void pay(double amount) const override {
        cout << "[Sistem] Metode Transfer dipilih. Silakan transfer Rp" << amount << " ke " << bankAccount << ".\n";
    }
};

// ==========================================
// 2. DATA MODELS (ENTITAS DATABASE UTAMA)
// ==========================================
class MenuItem {
public:
    string name;
    double price;
    string composition; 
    
    MenuItem() : name(""), price(0.0), composition("") {} // Perbaikan: Default Constructor
    MenuItem(string n, double p, string c) : name(n), price(p), composition(c) {}
};

class Order {
public:
    string orderId;
    string buyerName;
    string sellerName;
    double totalAmount;
    string status; 
    string pickupMethod; 
    string pickupTime;   
    unique_ptr<PaymentStrategy> paymentMethod;
    vector<MenuItem> items;

    Order(string id, string b, string s) 
        : orderId(id), buyerName(b), sellerName(s), totalAmount(0.0), status("Menunggu Konfirmasi Pembayaran") {}

    void addItem(const MenuItem& item) {
        items.push_back(item);
        totalAmount += item.price;
    }
    
    void setPaymentMethod(unique_ptr<PaymentStrategy> method) {
        paymentMethod = move(method);
    }
};

class Buyer {
public:
    string username;
    string password;
    string healthProfile; 
    
    Buyer() : username(""), password(""), healthProfile("Tidak ada alergi") {} // Perbaikan: Default Constructor
    Buyer(string u, string p) : username(u), password(p), healthProfile("Tidak ada alergi") {}
};

class Seller {
public:
    string username;
    string password;
    string storeName;
    bool isVerified; 
    vector<MenuItem> catalog;
    
    Seller() : username(""), password(""), storeName(""), isVerified(false) {} // Perbaikan: Default Constructor
    Seller(string u, string p, string store) : username(u), password(p), storeName(store), isVerified(false) {}
};

// IN-MEMORY DATABASE SIMULATION
map<string, Buyer> dbBuyers;
map<string, Seller> dbSellers;
vector<Order> dbOrders;
int orderCounter = 1;

// ==========================================
// 3. CQRS: DATA TRANSFER OBJECTS (DTO)
// ==========================================
struct OrderDTO {
    string orderId;
    double totalAmount;
    string status;
    string sellerName;
};

struct MenuDTO {
    string storeName;
    string itemName;
    double price;
    string composition;
};

// ==========================================
// 4. CQRS: QUERY SERVICE (READ ONLY)
// ==========================================
class QueryService {
public:
    bool isSellerVerified(const string& uname) {
        auto it = dbSellers.find(uname);
        return (it != dbSellers.end() && it->second.isVerified);
    }

    vector<MenuDTO> getRecommendations(const string& healthProfile) {
        vector<MenuDTO> result;
        for (const auto& pair : dbSellers) {
            if (!pair.second.isVerified) continue;
            for (const auto& item : pair.second.catalog) {
                if (healthProfile != "Tidak ada alergi" && item.composition.find(healthProfile) != string::npos) {
                    continue; 
                }
                result.push_back({pair.second.storeName, item.name, item.price, item.composition});
            }
        }
        return result;
    }

    vector<OrderDTO> getActiveOrders(const string& buyerUname) {
        vector<OrderDTO> result;
        for (const auto& o : dbOrders) {
            if (o.buyerName == buyerUname && o.status != "Selesai" && o.status != "Dibatalkan") {
                result.push_back({o.orderId, o.totalAmount, o.status, o.sellerName});
            }
        }
        return result;
    }

    vector<OrderDTO> getOrderHistory(const string& buyerUname) {
        vector<OrderDTO> result;
        for (const auto& o : dbOrders) {
            if (o.buyerName == buyerUname && (o.status == "Selesai" || o.status == "Dibatalkan")) {
                result.push_back({o.orderId, o.totalAmount, o.status, o.sellerName});
            }
        }
        return result;
    }

    vector<OrderDTO> getSellerOrders(const string& sellerUname) {
        vector<OrderDTO> result;
        for (const auto& o : dbOrders) {
            if (o.sellerName == sellerUname) {
                result.push_back({o.orderId, o.totalAmount, o.status, o.sellerName});
            }
        }
        return result;
    }
    
    int countNewOrders(const string& sellerUname) {
        int count = 0;
        for (const auto& o : dbOrders) {
            if (o.sellerName == sellerUname && o.status == "Menunggu Konfirmasi Penjual") count++;
        }
        return count;
    }
};

// ==========================================
// 5. CQRS: COMMAND SERVICE (WRITE ONLY)
// ==========================================
class CommandService {
public:
    bool verifySeller(const string& uname) {
        auto it = dbSellers.find(uname);
        if (it != dbSellers.end() && !it->second.isVerified) {
            it->second.isVerified = true;
            return true;
        }
        return false;
    }

    bool deleteSeller(const string& uname) {
        return dbSellers.erase(uname) > 0;
    }

    bool updateOrderStatus(const string& orderId, const string& sellerUname, const string& newStatus) {
        for (auto& o : dbOrders) {
            if (o.orderId == orderId && o.sellerName == sellerUname) {
                o.status = newStatus;
                return true;
            }
        }
        return false;
    }

    void saveOrder(Order order) {
        dbOrders.push_back(move(order));
    }
    
    void updateHealthProfile(const string& buyerUname, const string& newProfile) {
        dbBuyers.at(buyerUname).healthProfile = newProfile;
    }
    
    void updateStoreName(const string& sellerUname, const string& newName) {
        dbSellers.at(sellerUname).storeName = newName;
    }
};

// Instansiasi Global Services
QueryService queryService;
CommandService commandService;

// ==========================================
// 6. ANTARMUKA (UI) MENURUT AKTOR
// ==========================================

void runAdminMenu() {
    while (true) {
        cout << "\n=== DASHBOARD ADMIN ===\n";
        cout << "1. Verifikasi Penjual Baru\n";
        cout << "2. Hapus Data Penjual\n";
        cout << "3. Logout\n";
        cout << "Pilih menu: ";
        int choice = getValidIntInput();

        if (choice == 1) {
            cout << "\n--- Daftar Penjual Belum Terverifikasi ---\n";
            bool found = false;
            for (const auto& pair : dbSellers) {
                if (!pair.second.isVerified) {
                    cout << "- Username: " << pair.first << " | Toko: " << pair.second.storeName << "\n";
                    found = true;
                }
            }
            if (!found) { cout << "Semua penjual sudah diverifikasi.\n"; continue; }
            
            cout << "Ketik username penjual untuk diverifikasi: ";
            string uname; getline(cin, uname);
            if (commandService.verifySeller(uname)) cout << "[Sukses] Penjual diverifikasi!\n";
            else cout << "[Error] Username salah atau sudah diverifikasi.\n";

        } else if (choice == 2) {
            cout << "\n--- Daftar Seluruh Penjual ---\n";
            for (const auto& pair : dbSellers) {
                cout << "- Username: " << pair.first << " | Toko: " << pair.second.storeName << "\n";
            }
            cout << "Ketik username penjual yang akan dihapus: ";
            string uname; getline(cin, uname);
            if (commandService.deleteSeller(uname)) cout << "[Sukses] Data dihapus.\n";
            else cout << "[Error] Penjual tidak ditemukan.\n";

        } else if (choice == 3) break;
    }
}

void runSellerMenu(string sellerUname) {
    if (!queryService.isSellerVerified(sellerUname)) {
        cout << "\n[Peringatan] Akun Anda belum diverifikasi Admin. Harap tunggu!\n";
        return;
    }

    // UC08 Notifikasi
    int newOrders = queryService.countNewOrders(sellerUname);
    if (newOrders > 0) cout << "\n[NOTIFIKASI] Anda memiliki " << newOrders << " pesanan baru!\n";

    while (true) {
        Seller& currentSeller = dbSellers.at(sellerUname);
        cout << "\n=== TOKO: " << currentSeller.storeName << " ===\n";
        cout << "1. Kelola Katalog Menu\n";
        cout << "2. Ubah Status Pesanan\n";
        cout << "3. Atur Profil Penjual\n";
        cout << "4. Logout\n";
        cout << "Pilih menu: ";
        int choice = getValidIntInput();

        if (choice == 1) {
            cout << "\n1. Tambah Menu\n2. Hapus Menu\n3. Ubah Detail\nPilih: ";
            int menuChoice = getValidIntInput();
            
            if (menuChoice == 1) {
                string nama, comp; double harga;
                cout << "Nama Makanan: "; getline(cin, nama);
                cout << "Harga: "; harga = getValidIntInput();
                cout << "Komposisi (cth: ayam, cabai): "; getline(cin, comp);
                currentSeller.catalog.push_back(MenuItem(nama, harga, comp));
                cout << "[Sukses] Menu ditambahkan.\n";
            } else if (menuChoice == 2) {
                cout << "Ketik index menu (1-" << currentSeller.catalog.size() << ") untuk dihapus: ";
                int idx = getValidIntInput();
                if (idx > 0 && idx <= currentSeller.catalog.size()) {
                    currentSeller.catalog.erase(currentSeller.catalog.begin() + (idx - 1));
                    cout << "[Sukses] Menu dihapus.\n";
                }
            } else if (menuChoice == 3) {
                cout << "Ketik index menu (1-" << currentSeller.catalog.size() << ") untuk diubah: ";
                int idx = getValidIntInput();
                if (idx > 0 && idx <= currentSeller.catalog.size()) {
                    cout << "Nama Makanan Baru: "; getline(cin, currentSeller.catalog[idx-1].name);
                    cout << "Harga Baru: "; currentSeller.catalog[idx-1].price = getValidIntInput();
                    cout << "[Sukses] Detail diubah.\n";
                }
            }
        } else if (choice == 2) {
            vector<OrderDTO> orders = queryService.getSellerOrders(sellerUname);
            if (orders.empty()) { cout << "Belum ada pesanan.\n"; continue; }
            
            cout << "\n--- Daftar Pesanan ---\n";
            for (size_t i = 0; i < orders.size(); i++) {
                cout << i + 1 << ". ID: " << orders[i].orderId << " | Status: " << orders[i].status << "\n";
            }
            
            cout << "Pilih urutan pesanan (1-" << orders.size() << ") untuk diubah statusnya: ";
            int idx = getValidIntInput();
            if (idx > 0 && idx <= orders.size()) {
                cout << "Status Baru (1. Diproses, 2. Siap Diambil, 3. Selesai): ";
                int statChoice = getValidIntInput();
                string nStat = (statChoice == 1) ? "Sedang Diproses" : (statChoice == 2) ? "Siap Diambil/Diantar" : "Selesai";
                
                commandService.updateOrderStatus(orders[idx-1].orderId, sellerUname, nStat);
                cout << "[Sukses] Status diperbarui!\n";
            } else {
                cout << "[Error] Input tidak valid.\n";
            }
        } else if (choice == 3) {
            cout << "Nama Toko Baru: ";
            string newName; getline(cin, newName);
            commandService.updateStoreName(sellerUname, newName);
            cout << "[Sukses] Profil diperbarui.\n";
        } else if (choice == 4) break;
    }
}

void runBuyerMenu(string buyerUname) {
    while (true) {
        Buyer& currentBuyer = dbBuyers.at(buyerUname);
        cout << "\n=== HALAMAN PEMBELI (" << currentBuyer.username << ") ===\n";
        cout << "1. Buat Pesanan & Bayar\n";
        cout << "2. Lihat Status Pesanan Aktif\n";
        cout << "3. Lihat Rekomendasi Makanan\n";
        cout << "4. Lihat Riwayat Pesanan\n";
        cout << "5. Atur Form Kesehatan\n";
        cout << "6. Logout\n";
        cout << "Pilih menu: ";
        int choice = getValidIntInput();

        if (choice == 1) {
            cout << "\n--- Daftar Tempat Makan ---\n";
            vector<string> sellerKeys;
            for (auto& pair : dbSellers) {
                if (pair.second.isVerified && !pair.second.catalog.empty()) {
                    sellerKeys.push_back(pair.first);
                    cout << sellerKeys.size() << ". " << pair.second.storeName << "\n";
                }
            }
            if (sellerKeys.empty()) { cout << "Belum ada toko yang tersedia.\n"; continue; }
            
            cout << "Pilih nomor toko: ";
            int storeIdx = getValidIntInput() - 1;
            if (storeIdx < 0 || storeIdx >= sellerKeys.size()) continue;
            Seller& selectedSeller = dbSellers[sellerKeys[storeIdx]];

            string newOrderId = "ORD-TC-" + to_string(orderCounter++);
            Order newOrder(newOrderId, buyerUname, selectedSeller.username);

            bool ordering = true;
            while (ordering) {
                cout << "\n--- Menu " << selectedSeller.storeName << " ---\n";
                for (size_t i = 0; i < selectedSeller.catalog.size(); i++) {
                    cout << i + 1 << ". " << selectedSeller.catalog[i].name << " - Rp" << selectedSeller.catalog[i].price << "\n";
                }
                cout << selectedSeller.catalog.size() + 1 << ". Selesai & Lanjut Checkout\n";
                cout << selectedSeller.catalog.size() + 2 << ". Batal\n";
                cout << "Pilih: ";
                int menuIdx = getValidIntInput();

                if (menuIdx >= 1 && menuIdx <= selectedSeller.catalog.size()) {
                    cout << "Berapa porsi? "; int porsi = getValidIntInput();
                    for(int p=0; p<porsi; p++) {
                        newOrder.addItem(selectedSeller.catalog[menuIdx - 1]);
                    }
                    cout << "[+] " << porsi << " porsi ditambahkan.\n";
                } else if (menuIdx == selectedSeller.catalog.size() + 1) {
                    ordering = false;
                } else if (menuIdx == selectedSeller.catalog.size() + 2) {
                    cout << "Pesanan dibatalkan.\n";
                    ordering = false;
                    newOrder.totalAmount = 0; 
                }
            }

            if (newOrder.totalAmount > 0) {
                cout << "Waktu Pengambilan (cth: 12:00): "; getline(cin, newOrder.pickupTime);
                cout << "Metode (1. Ambil Sendiri, 2. Antar ke Plaza Supeno): "; 
                int method = getValidIntInput();
                newOrder.pickupMethod = (method == 1) ? "Ambil Sendiri" : "Diantar";

                cout << "\nTotal Tagihan: Rp" << newOrder.totalAmount << "\n";
                cout << "Metode Pembayaran (1. COD, 2. Transfer): ";
                int payChoice = getValidIntInput();
                if (payChoice == 1) {
                    newOrder.setPaymentMethod(make_unique<CODPayment>());
                } else {
                    newOrder.setPaymentMethod(make_unique<TransferPayment>("BSI 12345 a.n TC"));
                }
                newOrder.paymentMethod->pay(newOrder.totalAmount);
                newOrder.status = "Menunggu Konfirmasi Penjual";
                
                commandService.saveOrder(move(newOrder));
                cout << "[Sukses] Pesanan berhasil dibuat!\n";
            }

        } else if (choice == 2) {
            vector<OrderDTO> active = queryService.getActiveOrders(buyerUname);
            cout << "\n--- Status Pesanan Aktif ---\n";
            for (const auto& d : active) cout << "- ID: " << d.orderId << " | Toko: " << d.sellerName << " | Status: " << d.status << "\n";
            if(active.empty()) cout << "Tidak ada pesanan aktif.\n";

        } else if (choice == 3) {
            vector<MenuDTO> recs = queryService.getRecommendations(currentBuyer.healthProfile);
            cout << "\n--- Rekomendasi Menu Untuk Anda ---\n";
            cout << "(Memfilter alergi: " << currentBuyer.healthProfile << ")\n";
            for (const auto& d : recs) cout << "- " << d.itemName << " (Toko: " << d.storeName << ") - Rp" << d.price << "\n";
            if(recs.empty()) cout << "Tidak ada rekomendasi.\n";

        } else if (choice == 4) {
            vector<OrderDTO> history = queryService.getOrderHistory(buyerUname);
            cout << "\n--- Riwayat Pesanan ---\n";
            for (const auto& d : history) cout << "- ID: " << d.orderId << " | Status: " << d.status << "\n";
            if(history.empty()) cout << "Belum ada riwayat.\n";

        } else if (choice == 5) {
            cout << "Masukkan Profil Kesehatan (Alergi/Pantangan): ";
            string hp; getline(cin, hp);
            commandService.updateHealthProfile(buyerUname, hp);
            cout << "[Sukses] Profil kesehatan disimpan.\n";
        } else if (choice == 6) break;
    }
}

// ==========================================
// 7. SISTEM LOGIN MAIN MENU
// ==========================================
int main() {
    // Seed data Dummy dengan cara yang aman via map::operator[] karena sudah ada default constructor
    dbSellers["kantin_mbok"] = Seller("kantin_mbok", "123", "Kantin Mbok TC");
    dbSellers["kantin_mbok"].isVerified = true;
    dbSellers["kantin_mbok"].catalog.push_back(MenuItem("Ayam Geprek", 15000, "ayam, cabai"));
    dbSellers["kantin_mbok"].catalog.push_back(MenuItem("Nasi Goreng Seafood", 18000, "nasi, udang"));
    
    dbBuyers["mahasiswa1"] = Buyer("mahasiswa1", "123");

    while (true) {
        cout << "\n==========================================\n";
        cout << "      APLIKASI INFORMAKAN (LOGIN)\n";
        cout << "==========================================\n";
        cout << "1. Login Admin\n";
        cout << "2. Login Penjual\n";
        cout << "3. Login Pembeli\n";
        cout << "4. Daftar Penjual Baru\n";
        cout << "5. Keluar Aplikasi\n";
        cout << "Pilih: ";
        int choice = getValidIntInput();

        if (choice == 1) {
            cout << "Password Admin (ketik: admin): ";
            string p; getline(cin, p);
            if (p == "admin") runAdminMenu();
            else cout << "Password salah!\n";
        } else if (choice == 2) {
            cout << "Username Penjual: "; string u; getline(cin, u);
            if (dbSellers.count(u)) runSellerMenu(u);
            else cout << "Username tidak ditemukan.\n";
        } else if (choice == 3) {
            cout << "Username Pembeli (default: mahasiswa1): "; string u; getline(cin, u);
            if (!dbBuyers.count(u)) {
                cout << "Akun tidak ada, otomatis dibuatkan...\n";
                dbBuyers[u] = Buyer(u, "123");
            }
            runBuyerMenu(u);
        } else if (choice == 4) {
            cout << "Buat Username Penjual: "; string u; getline(cin, u);
            if (dbSellers.count(u)) { cout << "[Error] Username sudah terdaftar!\n"; continue; }
            cout << "Buat Password: "; string p; getline(cin, p);
            cout << "Nama Toko: "; string s; getline(cin, s);
            dbSellers[u] = Seller(u, p, s);
            cout << "[Sukses] Akun penjual didaftarkan. Menunggu verifikasi Admin.\n";
        } else if (choice == 5) {
            break;
        }
    }
    return 0;
}
```
