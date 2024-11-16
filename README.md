
# Aplikasi Pengelolaan Kontak

Sebuah aplikasi yang dapat digunakan untuk mengelola kontak menggunakan database SQLite, yang ditujukan untuk menyelesaikan Tugas PBO ke-6.

#### Source Code PengelolaanKontakFrame.java
```java
import java.io.*;
import java.sql.*;
import java.util.Vector;
import javax.swing.*;
import javax.swing.table.DefaultTableModel;
/**
 *
 * @author MyBook Z Series
 */
public class PengelolaanKontakFrame extends javax.swing.JFrame {

    private Connection connection;

    // Konstruktor untuk menginisialisasi komponen dan koneksi ke database
    public PengelolaanKontakFrame() {
        initComponents();
        connectDatabase();
        loadTableData();
        setupListeners();
        setupTableSelectionListener();
        setupListSelectionListener();
    }
    
    // Koneksi ke database
    private void connectDatabase() {
        try {
            connection = DriverManager.getConnection("jdbc:sqlite:kontak.db");
            Statement statement = connection.createStatement();
            statement.execute("CREATE TABLE IF NOT EXISTS kontak (" +
                    "id INTEGER PRIMARY KEY AUTOINCREMENT, " +
                    "nama TEXT NOT NULL, " +
                    "telepon TEXT NOT NULL, " +
                    "kategori TEXT NOT NULL)");
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Gagal terhubung ke database.", "Error", JOptionPane.ERROR_MESSAGE);
            e.printStackTrace();
        }
    }
    
    // Memuat data kontak dari database ke tabel
    private void loadTableData() {
        try {
            DefaultTableModel tableModel = (DefaultTableModel) jTable1.getModel();
            tableModel.setRowCount(0);  // Hapus data lama

            // Misalnya, kita ambil data dari database
            Connection conn = DriverManager.getConnection("jdbc:sqlite:kontak.db");
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT nama, telepon, kategori FROM kontak");

            while (rs.next()) {
                String nama = rs.getString("nama");
                String telepon = rs.getString("telepon");
                String kategori = rs.getString("kategori");
                tableModel.addRow(new Object[]{nama, telepon, kategori});
            }
            rs.close();
            stmt.close();
            conn.close();

            // Update JList setelah data dimuat di JTable
            updateListCariFromTable();
        } catch (SQLException ex) {
            JOptionPane.showMessageDialog(this, "Error loading data: " + ex.getMessage());
        }
    }
    
    // Menambahkan kontak baru ke database
    private void updateListCariFromTable() {
        DefaultListModel<String> listModel = new DefaultListModel<>();

        DefaultTableModel tableModel = (DefaultTableModel) jTable1.getModel();
        for (int i = 0; i < tableModel.getRowCount(); i++) {
            String nama = (String) tableModel.getValueAt(i, 0);  // Ambil nama dari kolom pertama
            listModel.addElement(nama);  // Tambahkan nama ke list
        }

        listCari.setModel(listModel);  // Set model ke listCari
    }
    
    // Menambah Action Listener untuk tombol-tombol
    private void setupListeners() {
        buttonTambah.addActionListener(e -> tambahKontak());
        buttonUbah.addActionListener(e -> ubahKontak());
        buttonHapus.addActionListener(e -> hapusKontak());
        buttonCari.addActionListener(e -> cariKontak());
        buttonImport.addActionListener(e -> importCSV());
        buttonExport.addActionListener(e -> exportCSV());
        
        buttonCari.addActionListener(e -> {
        String searchQuery = fieldCari.getText().trim();
        String kategori = (String) comboKategoriCari.getSelectedItem();
        filterTableData(searchQuery, kategori);
        });
    }
    
    // Menambahkan kontak baru ke database
    private void tambahKontak() {
        String nama = fieldNama.getText().trim();
        String telepon = fieldTelepon.getText().trim();
        String kategori = comboKategori.getSelectedItem().toString();

        if (nama.isEmpty() || telepon.isEmpty() || kategori.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Semua field harus diisi.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }

        if (!telepon.matches("\\d{10,13}")) {
            JOptionPane.showMessageDialog(this, "Nomor telepon harus berupa angka 10-13 digit.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }
        
        // Periksa apakah nama sudah ada di database
        if (isNamaExist(nama)) {
            JOptionPane.showMessageDialog(this, "Kontak dengan nama " + nama + " sudah ada.");
            return;
        }

        try {
            String sql = "INSERT INTO kontak (nama, telepon, kategori) VALUES (?, ?, ?)";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, nama);
            preparedStatement.setString(2, telepon);
            preparedStatement.setString(3, kategori);
            preparedStatement.executeUpdate();

            loadTableData();
            clearForm();
            JOptionPane.showMessageDialog(this, "Kontak berhasil ditambahkan.", "Informasi", JOptionPane.INFORMATION_MESSAGE);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // Cek nama sudah ada di database atau tidak
    private boolean isNamaExist(String nama) {
        try {
            Connection conn = DriverManager.getConnection("jdbc:sqlite:kontak.db");
            PreparedStatement stmt = conn.prepareStatement("SELECT COUNT(*) FROM kontak WHERE nama = ?");
            stmt.setString(1, nama);
            ResultSet rs = stmt.executeQuery();
            int count = rs.getInt(1);
            rs.close();
            stmt.close();
            conn.close();

            return count > 0;  // Jika lebih dari 0 berarti nama sudah ada
        } catch (SQLException ex) {
            JOptionPane.showMessageDialog(this, "Error checking name existence: " + ex.getMessage());
            return false;
        }
    }
    
    // Mengubah kontak yang sudah ada
    private void ubahKontak() {
        int selectedRow = jTable1.getSelectedRow();
        if (selectedRow == -1) {
            JOptionPane.showMessageDialog(this, "Pilih kontak yang ingin diubah.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }

        String nama = fieldNama.getText().trim();
        String telepon = fieldTelepon.getText().trim();
        String kategori = comboKategori.getSelectedItem().toString();

        if (nama.isEmpty() || telepon.isEmpty() || kategori.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Semua field harus diisi.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }

        if (!telepon.matches("\\d{10,13}")) {
            JOptionPane.showMessageDialog(this, "Nomor telepon harus berupa angka 10-13 digit.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }

        try {
            String selectedNama = (String) jTable1.getValueAt(selectedRow, 0);

            String sql = "UPDATE kontak SET nama = ?, telepon = ?, kategori = ? WHERE nama = ?";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, nama);
            preparedStatement.setString(2, telepon);
            preparedStatement.setString(3, kategori);
            preparedStatement.setString(4, selectedNama);
            preparedStatement.executeUpdate();

            loadTableData();
            clearForm();
            JOptionPane.showMessageDialog(this, "Kontak berhasil diubah.", "Informasi", JOptionPane.INFORMATION_MESSAGE);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    //Menghapus kontak yang sudah ada
    private void hapusKontak() {
        int selectedRow = jTable1.getSelectedRow();
        if (selectedRow == -1) {
            JOptionPane.showMessageDialog(this, "Pilih kontak yang ingin dihapus.", "Peringatan", JOptionPane.WARNING_MESSAGE);
            return;
        }

        String selectedNama = (String) jTable1.getValueAt(selectedRow, 0);

        try {
            String sql = "DELETE FROM kontak WHERE nama = ?";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, selectedNama);
            preparedStatement.executeUpdate();

            loadTableData();
            clearForm();
            JOptionPane.showMessageDialog(this, "Kontak berhasil dihapus.", "Informasi", JOptionPane.INFORMATION_MESSAGE);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    //Mencari kontak dan menampilkannya ke tabel
    private void cariKontak() {
        String cari = fieldCari.getText().trim();
        String kategori = comboKategoriCari.getSelectedItem().toString();

        try {
            String sql = "SELECT nama, telepon, kategori FROM kontak WHERE (nama LIKE ? OR telepon LIKE ?)";
            if (!kategori.equals("Semua")) {
                sql += " AND kategori = ?";
            }

            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, "%" + cari + "%");
            preparedStatement.setString(2, "%" + cari + "%");

            if (!kategori.equals("Semua")) {
                preparedStatement.setString(3, kategori);
            }

            ResultSet resultSet = preparedStatement.executeQuery();

            DefaultTableModel model = (DefaultTableModel) jTable1.getModel();
            model.setRowCount(0);

            while (resultSet.next()) {
                Vector<String> row = new Vector<>();
                row.add(resultSet.getString("nama"));
                row.add(resultSet.getString("telepon"));
                row.add(resultSet.getString("kategori"));
                model.addRow(row);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // Mengimpor data dari file CSV
    private void importCSV() {
        JFileChooser fileChooser = new JFileChooser();
        if (fileChooser.showOpenDialog(this) == JFileChooser.APPROVE_OPTION) {
            File file = fileChooser.getSelectedFile();
            try (BufferedReader br = new BufferedReader(new FileReader(file))) {
                String line;
                while ((line = br.readLine()) != null) {
                    String[] data = line.split(",");
                    String sql = "INSERT INTO kontak (nama, telepon, kategori) VALUES (?, ?, ?)";
                    PreparedStatement preparedStatement = connection.prepareStatement(sql);
                    preparedStatement.setString(1, data[0]);
                    preparedStatement.setString(2, data[1]);
                    preparedStatement.setString(3, data[2]);
                    preparedStatement.executeUpdate();
                }
                loadTableData();
                JOptionPane.showMessageDialog(this, "Data berhasil diimpor.", "Informasi", JOptionPane.INFORMATION_MESSAGE);
            } catch (Exception e) {
                e.printStackTrace();
                JOptionPane.showMessageDialog(this, "Gagal mengimpor data.", "Error", JOptionPane.ERROR_MESSAGE);
            }
        }
    }
    
    // Menyaring data berdasarkan pencarian dan kategori
    private void filterTableData(String searchQuery, String kategori) {
        DefaultTableModel model = (DefaultTableModel) jTable1.getModel();
        DefaultListModel<String> listModel = new DefaultListModel<>();

        // Menghapus semua data yang ada di JList sebelum memperbarui
        listModel.clear();

        for (int i = 0; i < model.getRowCount(); i++) {
            String nama = (String) model.getValueAt(i, 0);
            String telepon = (String) model.getValueAt(i, 1);
            String kat = (String) model.getValueAt(i, 2);

            // Menyesuaikan filter berdasarkan kategori dan pencarian nama/telepon
            if ((kategori.equals("Semua") || kat.equals(kategori)) &&
                (nama.toLowerCase().contains(searchQuery.toLowerCase()) || 
                telepon.contains(searchQuery))) {

                // Menambahkan nama ke JList jika cocok dengan pencarian
                listModel.addElement(nama);
            }
        }

        // Set model baru ke JList
        listCari.setModel(listModel);
    }
    
    // Mengekspor data menjadi file CSV
    private void exportCSV() {
        JFileChooser fileChooser = new JFileChooser();
        if (fileChooser.showSaveDialog(this) == JFileChooser.APPROVE_OPTION) {
            File file = fileChooser.getSelectedFile();
            try (BufferedWriter bw = new BufferedWriter(new FileWriter(file))) {
                DefaultTableModel model = (DefaultTableModel) jTable1.getModel();
                for (int i = 0; i < model.getRowCount(); i++) {
                    for (int j = 0; j < model.getColumnCount(); j++) {
                        bw.write(model.getValueAt(i, j).toString());
                        if (j < model.getColumnCount() - 1) {
                            bw.write(",");
                        }
                    }
                    bw.newLine();
                }
                JOptionPane.showMessageDialog(this, "Data berhasil diekspor.", "Informasi", JOptionPane.INFORMATION_MESSAGE);
            } catch (IOException e) {
                e.printStackTrace();
                JOptionPane.showMessageDialog(this, "Gagal mengekspor data.", "Error", JOptionPane.ERROR_MESSAGE);
            }
        }
    }
    
    // Setup listener untuk pemilihan di tabel
    private void setupTableSelectionListener() {
        jTable1.getSelectionModel().addListSelectionListener(e -> {
            if (!e.getValueIsAdjusting()) {
                int selectedRow = jTable1.getSelectedRow();
                if (selectedRow != -1) {
                    String selectedNama = (String) jTable1.getValueAt(selectedRow, 0);

                    // Cari kontak di listCari berdasarkan nama yang dipilih
                    DefaultListModel<String> listModel = (DefaultListModel<String>) listCari.getModel();
                    for (int i = 0; i < listModel.size(); i++) {
                        if (listModel.getElementAt(i).equals(selectedNama)) {
                            // Pilih item yang sesuai di JList
                            listCari.setSelectedIndex(i);
                            break;
                        }
                    }

                    // Set data ke form
                    String telepon = (String) jTable1.getValueAt(selectedRow, 1);
                    String kategori = (String) jTable1.getValueAt(selectedRow, 2);
                    fieldNama.setText(selectedNama);
                    fieldTelepon.setText(telepon);
                    comboKategori.setSelectedItem(kategori);
                }
            }
        });
    }
    // Setup listener untuk pemilihan di list pencarian
    private void setupListSelectionListener() {
        listCari.addListSelectionListener(e -> {
            if (!e.getValueIsAdjusting()) {
                String selectedNama = listCari.getSelectedValue();
                if (selectedNama != null) {
                    // Cari kontak di tabel berdasarkan nama
                    DefaultTableModel model = (DefaultTableModel) jTable1.getModel();
                    for (int i = 0; i < model.getRowCount(); i++) {
                        String nama = (String) model.getValueAt(i, 0);
                        if (nama.equals(selectedNama)) {
                            // Pilih baris yang sesuai di jTable1
                            jTable1.setRowSelectionInterval(i, i);

                            // Set data ke form
                            String telepon = (String) model.getValueAt(i, 1);
                            String kategori = (String) model.getValueAt(i, 2);
                            fieldNama.setText(nama);
                            fieldTelepon.setText(telepon);
                            comboKategori.setSelectedItem(kategori);
                            break;
                        }
                    }
                }
            }
        });
    }
    
    //Membersihkan form input
    private void clearForm() {
        fieldNama.setText("");
        fieldTelepon.setText("");
        comboKategori.setSelectedIndex(0);
    }
    /**
     * This method is called from within the constructor to initialize the form.
     * WARNING: Do NOT modify this code. The content of this method is always
     * regenerated by the Form Editor.
     */
    @SuppressWarnings("unchecked")
    // <editor-fold defaultstate="collapsed" desc="Generated Code">                          
    private void initComponents() {
        java.awt.GridBagConstraints gridBagConstraints;

        JOptionPane = new javax.swing.JOptionPane();
        jLabel1 = new javax.swing.JLabel();
        panelCRUD = new javax.swing.JPanel();
        jLabel2 = new javax.swing.JLabel();
        jLabel3 = new javax.swing.JLabel();
        jLabel4 = new javax.swing.JLabel();
        fieldNama = new javax.swing.JTextField();
        fieldTelepon = new javax.swing.JTextField();
        comboKategori = new javax.swing.JComboBox<>();
        jPanel2 = new javax.swing.JPanel();
        buttonTambah = new javax.swing.JButton();
        buttonUbah = new javax.swing.JButton();
        buttonHapus = new javax.swing.JButton();
        panelPencarian = new javax.swing.JPanel();
        jScrollPane2 = new javax.swing.JScrollPane();
        listCari = new javax.swing.JList<>();
        fieldCari = new javax.swing.JTextField();
        jLabel5 = new javax.swing.JLabel();
        buttonCari = new javax.swing.JButton();
        comboKategoriCari = new javax.swing.JComboBox<>();
        panelTabel = new javax.swing.JPanel();
        jScrollPane1 = new javax.swing.JScrollPane();
        jTable1 = new javax.swing.JTable();
        panelButton = new javax.swing.JPanel();
        buttonImport = new javax.swing.JButton();
        buttonExport = new javax.swing.JButton();

        setDefaultCloseOperation(javax.swing.WindowConstants.EXIT_ON_CLOSE);
        getContentPane().setLayout(new java.awt.GridBagLayout());

        jLabel1.setFont(new java.awt.Font("Segoe UI Variable", 1, 18)); // NOI18N
        jLabel1.setText("Aplikasi Pengelolaan Kontak");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 0;
        gridBagConstraints.gridwidth = 2;
        gridBagConstraints.insets = new java.awt.Insets(0, 0, 14, 0);
        getContentPane().add(jLabel1, gridBagConstraints);

        panelCRUD.setBackground(new java.awt.Color(255, 204, 204));
        panelCRUD.setBorder(javax.swing.BorderFactory.createBevelBorder(javax.swing.border.BevelBorder.RAISED));
        panelCRUD.setLayout(new java.awt.GridBagLayout());

        jLabel2.setText("Nama");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 1;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_START;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(jLabel2, gridBagConstraints);

        jLabel3.setText("Telepon");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_START;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(jLabel3, gridBagConstraints);

        jLabel4.setText("Kategori");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 3;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_START;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(jLabel4, gridBagConstraints);
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 1;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.ipadx = 400;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_END;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(fieldNama, gridBagConstraints);
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.ipadx = 200;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_END;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(fieldTelepon, gridBagConstraints);

        comboKategori.setModel(new javax.swing.DefaultComboBoxModel<>(new String[] { "Keluarga", "Teman", "Kerja" }));
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 3;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.LINE_END;
        gridBagConstraints.insets = new java.awt.Insets(7, 20, 7, 20);
        panelCRUD.add(comboKategori, gridBagConstraints);

        jPanel2.setBackground(new java.awt.Color(255, 204, 204));
        jPanel2.setLayout(new java.awt.GridBagLayout());

        buttonTambah.setText("Tambah");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.insets = new java.awt.Insets(0, 2, 0, 2);
        jPanel2.add(buttonTambah, gridBagConstraints);

        buttonUbah.setText("Ubah");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 0;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.insets = new java.awt.Insets(0, 2, 0, 2);
        jPanel2.add(buttonUbah, gridBagConstraints);

        buttonHapus.setText("Hapus");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 2;
        gridBagConstraints.gridy = 0;
        gridBagConstraints.insets = new java.awt.Insets(0, 2, 0, 2);
        jPanel2.add(buttonHapus, gridBagConstraints);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 4;
        gridBagConstraints.insets = new java.awt.Insets(7, 0, 7, 0);
        panelCRUD.add(jPanel2, gridBagConstraints);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 1;
        getContentPane().add(panelCRUD, gridBagConstraints);

        panelPencarian.setBackground(new java.awt.Color(153, 153, 255));
        panelPencarian.setBorder(javax.swing.BorderFactory.createBevelBorder(javax.swing.border.BevelBorder.RAISED));
        panelPencarian.setLayout(new java.awt.GridBagLayout());

        jScrollPane2.setViewportView(listCari);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 3;
        gridBagConstraints.gridwidth = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.BOTH;
        gridBagConstraints.ipady = 200;
        panelPencarian.add(jScrollPane2, gridBagConstraints);
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 1;
        gridBagConstraints.ipadx = 150;
        panelPencarian.add(fieldCari, gridBagConstraints);

        jLabel5.setFont(new java.awt.Font("Segoe UI Variable", 1, 14)); // NOI18N
        jLabel5.setText("Pencarian");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 0;
        gridBagConstraints.gridwidth = 2;
        panelPencarian.add(jLabel5, gridBagConstraints);

        buttonCari.setText("Cari");
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 1;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        panelPencarian.add(buttonCari, gridBagConstraints);

        comboKategoriCari.setModel(new javax.swing.DefaultComboBoxModel<>(new String[] { "Semua", "Keluarga", "Teman", "Kerja" }));
        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 2;
        gridBagConstraints.gridwidth = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        panelPencarian.add(comboKategoriCari, gridBagConstraints);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 1;
        gridBagConstraints.gridy = 1;
        gridBagConstraints.gridheight = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.VERTICAL;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.PAGE_START;
        getContentPane().add(panelPencarian, gridBagConstraints);

        panelTabel.setBackground(new java.awt.Color(204, 255, 204));
        panelTabel.setBorder(javax.swing.BorderFactory.createBevelBorder(javax.swing.border.BevelBorder.RAISED));
        panelTabel.setLayout(new java.awt.GridBagLayout());

        jTable1.setModel(new javax.swing.table.DefaultTableModel(
            new Object [][] {
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null},
                {null, null, null}
            },
            new String [] {
                "Nama", "Telepon", "Kategori"
            }
        ) {
            Class[] types = new Class [] {
                java.lang.String.class, java.lang.String.class, java.lang.String.class
            };

            public Class getColumnClass(int columnIndex) {
                return types [columnIndex];
            }
        });
        jScrollPane1.setViewportView(jTable1);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 0;
        gridBagConstraints.fill = java.awt.GridBagConstraints.BOTH;
        gridBagConstraints.ipadx = -429;
        gridBagConstraints.ipady = -402;
        gridBagConstraints.anchor = java.awt.GridBagConstraints.NORTHWEST;
        gridBagConstraints.weightx = 1.0;
        gridBagConstraints.weighty = 1.0;
        panelTabel.add(jScrollPane1, gridBagConstraints);

        panelButton.setBackground(new java.awt.Color(204, 255, 204));

        buttonImport.setText("Import");
        panelButton.add(buttonImport);

        buttonExport.setText("Export");
        panelButton.add(buttonExport);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 1;
        panelTabel.add(panelButton, gridBagConstraints);

        gridBagConstraints = new java.awt.GridBagConstraints();
        gridBagConstraints.gridx = 0;
        gridBagConstraints.gridy = 2;
        gridBagConstraints.fill = java.awt.GridBagConstraints.HORIZONTAL;
        gridBagConstraints.ipady = 200;
        getContentPane().add(panelTabel, gridBagConstraints);

        pack();
    }// </editor-fold>                        

    /**
     * @param args the command line arguments
     */
    public static void main(String args[]) {
        /* Set the Nimbus look and feel */
        //<editor-fold defaultstate="collapsed" desc=" Look and feel setting code (optional) ">
        /* If Nimbus (introduced in Java SE 6) is not available, stay with the default look and feel.
         * For details see http://download.oracle.com/javase/tutorial/uiswing/lookandfeel/plaf.html 
         */
        try {
            for (javax.swing.UIManager.LookAndFeelInfo info : javax.swing.UIManager.getInstalledLookAndFeels()) {
                if ("Nimbus".equals(info.getName())) {
                    javax.swing.UIManager.setLookAndFeel(info.getClassName());
                    break;
                }
            }
        } catch (ClassNotFoundException ex) {
            java.util.logging.Logger.getLogger(PengelolaanKontakFrame.class.getName()).log(java.util.logging.Level.SEVERE, null, ex);
        } catch (InstantiationException ex) {
            java.util.logging.Logger.getLogger(PengelolaanKontakFrame.class.getName()).log(java.util.logging.Level.SEVERE, null, ex);
        } catch (IllegalAccessException ex) {
            java.util.logging.Logger.getLogger(PengelolaanKontakFrame.class.getName()).log(java.util.logging.Level.SEVERE, null, ex);
        } catch (javax.swing.UnsupportedLookAndFeelException ex) {
            java.util.logging.Logger.getLogger(PengelolaanKontakFrame.class.getName()).log(java.util.logging.Level.SEVERE, null, ex);
        }
        //</editor-fold>

        /* Create and display the form */
        java.awt.EventQueue.invokeLater(new Runnable() {
            public void run() {
                new PengelolaanKontakFrame().setVisible(true);
            }
        });
    }

    // Variables declaration - do not modify                     
    private javax.swing.JOptionPane JOptionPane;
    private javax.swing.JButton buttonCari;
    private javax.swing.JButton buttonExport;
    private javax.swing.JButton buttonHapus;
    private javax.swing.JButton buttonImport;
    private javax.swing.JButton buttonTambah;
    private javax.swing.JButton buttonUbah;
    private javax.swing.JComboBox<String> comboKategori;
    private javax.swing.JComboBox<String> comboKategoriCari;
    private javax.swing.JTextField fieldCari;
    private javax.swing.JTextField fieldNama;
    private javax.swing.JTextField fieldTelepon;
    private javax.swing.JLabel jLabel1;
    private javax.swing.JLabel jLabel2;
    private javax.swing.JLabel jLabel3;
    private javax.swing.JLabel jLabel4;
    private javax.swing.JLabel jLabel5;
    private javax.swing.JPanel jPanel2;
    private javax.swing.JScrollPane jScrollPane1;
    private javax.swing.JScrollPane jScrollPane2;
    private javax.swing.JTable jTable1;
    private javax.swing.JList<String> listCari;
    private javax.swing.JPanel panelButton;
    private javax.swing.JPanel panelCRUD;
    private javax.swing.JPanel panelPencarian;
    private javax.swing.JPanel panelTabel;
    // End of variables declaration                   
}
```
## Fitur Utama
- Menyimpan informasi kontak menggunakan databse SQLite
- Menambah, mengedit, menghapus kontak tersimpan
- Pengelompokan berdasarkan kategori

## Fitur Tambahan (Variasi)
- Fitur Pencarian berdasarkan nama, nomor, dan kategori
- Validasi input untuk memastikan nomor telepon yang dimasukan hanya berisi angka dan memiliki panjang yang sesuai
- Fitur untuk mengekspor database ke file CSV dan mengimpor kontak dari file tersebut ke database.
## Screenshots
![{3925E241-E654-4405-9D28-6CCFC9766302}](https://github.com/user-attachments/assets/185a58c0-5513-46c0-a8ec-ebc376fce46a)



## Referensi

 - [Modul PBO2 Latihan 3](https://drive.google.com/file/d/107UW9UZi2DFzhNVmcyrvKAHtSVj4cYVt/view)


## Biodata Pembuat
- Nama : Bintang Putra Setiawan
- Kelas : 5B Reg Pagi Banjarmasin
- NPM : 2210010539
- [@Bintangps01](https://github.com/Bintangps01)
