

    import java.io.*;
    import java.time.LocalDate;
    import java.time.format.DateTimeFormatter;
    import java.util.*;

    class Book {
    int id;
    String title, author;
    boolean isIssued;

    Book(int id, String title, String author, boolean isIssued) {
        this.id = id;
        this.title = title;
        this.author = author;
        this.isIssued = isIssued;
    }

    public String toCSV() {
        return id + "," + title + "," + author + "," + isIssued;
    }

    public String toString() {
        return "ID: " + id + ", Title: " + title + ", Author: " + author + ", Issued: " + isIssued;
    }
    }

    class User {
    String username;
    String role;

    User(String username, String role) {
        this.username = username;
        this.role = role;
    }
    }

    public class LibrarySystem {
    static Scanner sc = new Scanner(System.in);
    static List<Book> books = new ArrayList<>();
    static final String BOOKS_FILE = "books.csv";
    static final String USERS_FILE = "users.csv";
    static final String ISSUED_FILE = "issued.csv";
    static DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    public static void main(String[] args) throws IOException {
        loadBooks();
        User user = login();

        if (user == null) return;

        if (user.role.equals("admin")) adminMenu();
        else userMenu(user.username);

        saveBooks();
    }

    static User login() {
        System.out.print("Username: ");
        String uname = sc.nextLine();
        System.out.print("Password: ");
        String pwd = sc.nextLine();

        try (BufferedReader br = new BufferedReader(new FileReader(USERS_FILE))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] parts = line.split(",");
                if (parts[0].equals(uname) && parts[1].equals(pwd)) {
                    System.out.println("✅ Login successful as " + parts[2]);
                    return new User(uname, parts[2]);
                }
            }
        } catch (IOException e) {
            System.out.println("❌ Error reading users file: " + e.getMessage());
        }

        System.out.println("❌ Invalid credentials.");
        return null;
    }

    static void loadBooks() {
        File file = new File(BOOKS_FILE);
        if (!file.exists()) return;

        try (BufferedReader br = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] b = line.split(",");
                books.add(new Book(Integer.parseInt(b[0]), b[1], b[2], Boolean.parseBoolean(b[3])));
            }
        } catch (IOException e) {
            System.out.println("❌ Error reading books file: " + e.getMessage());
        }
    }

    static void saveBooks() {
        try (BufferedWriter bw = new BufferedWriter(new FileWriter(BOOKS_FILE))) {
            for (Book b : books) {
                bw.write(b.toCSV());
                bw.newLine();
            }
        } catch (IOException e) {
            System.out.println("❌ Error saving books: " + e.getMessage());
        }
    }

    static void adminMenu() {
        int choice;
        do {
            System.out.println("\n--- Admin Menu ---");
            System.out.println("1. Add Book");
            System.out.println("2. View Books");
            System.out.println("3. Exit");
            System.out.print("Choice: ");
            choice = Integer.parseInt(sc.nextLine());

            switch (choice) {
                case 1 -> addBook();
                case 2 -> viewBooks();
                case 3 -> System.out.println("👋 Exiting...");
                default -> System.out.println("❌ Invalid choice");
            }
        } while (choice != 3);
    }

    static void userMenu(String username) {
        int choice;
        do {
            System.out.println("\n--- User Menu ---");
            System.out.println("1. View Books");
            System.out.println("2. Issue Book");
            System.out.println("3. Return Book");
            System.out.println("4. Exit");
            System.out.print("Choice: ");
            choice = Integer.parseInt(sc.nextLine());

            switch (choice) {
                case 1 -> viewBooks();
                case 2 -> issueBook(username);
                case 3 -> returnBook(username);
                case 4 -> System.out.println("👋 Exiting...");
                default -> System.out.println("❌ Invalid choice");
            }
        } while (choice != 4);
    }

    static void addBook() {
        System.out.print("Book ID: ");
        int id = Integer.parseInt(sc.nextLine());
        System.out.print("Title: ");
        String title = sc.nextLine();
        System.out.print("Author: ");
        String author = sc.nextLine();

        books.add(new Book(id, title, author, false));
        System.out.println("✅ Book added.");
    }

    static void viewBooks() {
        if (books.isEmpty()) {
            System.out.println("📚 No books available.");
            return;
        }
        for (Book b : books) {
            System.out.println(b);
        }
    }

    static void issueBook(String username) {
        System.out.print("Enter book ID to issue: ");
        int id = Integer.parseInt(sc.nextLine());
        for (Book b : books) {
            if (b.id == id) {
                if (b.isIssued) {
                    System.out.println("⚠️ Book is already issued.");
                    return;
                }
                b.isIssued = true;

                String dueDate = LocalDate.now().plusDays(7).format(formatter);
                try (BufferedWriter bw = new BufferedWriter(new FileWriter(ISSUED_FILE, true))) {
                    bw.write(username + "," + id + "," + dueDate);
                    bw.newLine();
                } catch (IOException e) {
                    System.out.println("❌ Error writing to issued file: " + e.getMessage());
                }

                System.out.println("✅ Book issued. Due date: " + dueDate);
                return;
            }
        }
        System.out.println("❌ Book not found.");
    }

    static void returnBook(String username) {
        System.out.print("Enter book ID to return: ");
        int id = Integer.parseInt(sc.nextLine());

        File inputFile = new File(ISSUED_FILE);
        File tempFile = new File("temp.csv");

        boolean found = false;

        try (BufferedReader br = new BufferedReader(new FileReader(inputFile));
             BufferedWriter bw = new BufferedWriter(new FileWriter(tempFile))) {

            String line;
            while ((line = br.readLine()) != null) {
                String[] parts = line.split(",");
                if (parts[0].equals(username) && Integer.parseInt(parts[1]) == id) {
                    found = true;
                    for (Book b : books) {
                        if (b.id == id) b.isIssued = false;
                    }
                    continue; // Skip this line (i.e., remove from issued list)
                }
                bw.write(line);
                bw.newLine();
            }

        } catch (IOException e) {
            System.out.println("❌ Error processing return: " + e.getMessage());
        }

        if (found) {
            inputFile.delete();
            tempFile.renameTo(inputFile);
            System.out.println("✅ Book returned.");
        } else {
            System.out.println("❌ No such issued book found.");
        }
    }
    }
