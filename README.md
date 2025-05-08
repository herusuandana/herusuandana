/**
 * HeruPedia Auto Updater - Frontend Script
 *
 * Menangani antarmuka pengguna untuk sistem update otomatis
 * dengan dukungan real-time logging dan monitoring status
 *
 * @author Heru Suandana
 * @version 2.0
 */

class HeruUpdater {
    constructor(options) {
        // Opsi konfigurasi
        this.options = Object.assign({
            updateEndpoint: BASE_URL + 'admin/updater/jalankan',
            statusEndpoint: BASE_URL + 'admin/updater/get_status',
            logEndpoint: BASE_URL + 'admin/updater/stream_log',
            cancelEndpoint: BASE_URL + 'admin/updater/cancel_update',
            updateInterval: 1000, // Interval polling untuk status update
            maxRetries: 3,        // Jumlah percobaan maksimal untuk request AJAX
        }, options);

        // Elemen UI
        this.elements = {
            startButton: document.getElementById('btnMulaiUpdate'),
            cancelButton: document.getElementById('btnCancelUpdate'),
            statusBox: document.getElementById('statusBox'),
            progressBar: document.getElementById('progressBar'),
            progressValue: document.getElementById('progressValue'),
            logConsole: document.getElementById('logConsole'),
            loadingIcon: document.getElementById('loadingIcon'),
            updateModal: document.getElementById('modalUpdate')
        };

        // Inisialisasi EventSource untuk log streaming
        this.eventSource = null;

        // Status interval polling
        this.statusInterval = null;

        // Indikator apakah update sedang berjalan
        this.isUpdating = false;

        // Event handlers
        this.setupEventHandlers();
    }

    /**
     * Setup event handlers untuk UI
     */
    setupEventHandlers() {
        // Handler untuk tombol mulai update
        if (this.elements.startButton) {
            this.elements.startButton.addEventListener('click', () => this.startUpdate());
        }

        // Handler untuk tombol cancel update
        if (this.elements.cancelButton) {
            this.elements.cancelButton.addEventListener('click', () => this.cancelUpdate());
        }

        // Handler untuk menutup modal
        if (this.elements.updateModal) {
            this.elements.updateModal.addEventListener('hidden.bs.modal', () => {
                if (this.isUpdating) {
                    // Jika update masih berjalan, tampilkan konfirmasi
                    if (confirm('Update masih berjalan. Yakin ingin menutup jendela ini?')) {
                        this.stopMonitoring();
                    } else {
                        // Buka kembali modal
                        const modal = bootstrap.Modal.getInstance(this.elements.updateModal);
                        modal.show();
                    }
                } else {
                    this.stopMonitoring();
                }
            });
        }
    }

    /**
     * Mulai proses update
     */
    startUpdate() {
        // Set UI ke mode loading
        this.setUILoading(true);

        // Set status update sedang berjalan
        this.isUpdating = true;

        // Reset log console
        if (this.elements.logConsole) {
            this.elements.logConsole.innerHTML = '';
            this.elements.logConsole.classList.remove('d-none');
        }

        // Tampilkan status box
        if (this.elements.statusBox) {
            this.elements.statusBox.className = 'alert alert-info';
            this.elements.statusBox.classList.remove('d-none');
            this.elements.statusBox.innerText = 'Memulai proses pembaruan...';
        }

        // Tampilkan progress bar
        if (this.elements.progressBar) {
            this.elements.progressBar.classList.remove('d-none');
        }

        // Mulai streaming log
        this.startLogStream();

        // Mulai polling status
        this.startStatusPolling();

        // Kirim request untuk memulai update
        fetch(this.options.updateEndpoint)
            .then(res => res.json())
            .then(data => this.handleUpdateResponse(data))
            .catch(err => this.handleUpdateError(err));
    }

    /**
     * Handle response dari endpoint update
     */
    handleUpdateResponse(data) {
        if (data.status === 'success') {
            // Update berhasil
            this.addStatusMessage('success', data.message);

            // Update button menjadi reload
            if (this.elements.startButton) {
                this.elements.startButton.classList.replace('btn-primary', 'btn-success');
                this.elements.startButton.innerHTML = "<i class='fas fa-check me-2'></i> Pembaruan Selesai - Muat Ulang";
                this.elements.startButton.disabled = false;
                this.elements.startButton.onclick = () => location.reload();
            }

            // Sembunyikan tombol cancel
            if (this.elements.cancelButton) {
                this.elements.cancelButton.classList.add('d-none');
            }

            // Set status selesai update
            this.isUpdating = false;

        } else if (data.status === 'info') {
            // Informasi saja
            this.addStatusMessage('info', data.message);
            this.setUILoading(false);
            this.isUpdating = false;

        } else {
            // Error
            this.addStatusMessage('danger', data.message || 'Terjadi kesalahan yang tidak diketahui');
            this.setUILoading(false);
            this.isUpdating = false;
        }
    }

    /**
     * Handle error saat melakukan request update
     */
    handleUpdateError(err) {
        console.error('Update error:', err);
        this.addStatusMessage('danger', 'Gagal menjalankan update: ' + err.message);
        this.setUILoading(false);
        this.isUpdating = false;
    }

    /**
     * Batalkan update yang sedang berjalan
     */
    cancelUpdate() {
        if (!this.isUpdating) return;

        if (confirm('Yakin ingin membatalkan proses update? Ini dapat menyebabkan sistem tidak stabil.')) {
            // Kirim request untuk membatalkan update
            fetch(this.options.cancelEndpoint)
                .then(res => res.json())
                .then(data => {
                    this.addStatusMessage(data.status === 'success' ? 'warning' : 'danger', data.message);
                    this.setUILoading(false);
                    this.isUpdating = false;
                })
                .catch(err => {
                    console.error('Cancel error:', err);
                    this.addStatusMessage('danger', 'Gagal membatalkan update: ' + err.message);
                });
        }
    }

    /**
     * Mulai streaming log secara real-time
     */
    startLogStream() {
        // Hentikan EventSource yang ada jika ada
        if (this.eventSource) {
            this.eventSource.close();
        }

        // Buat EventSource baru
        this.eventSource = new EventSource(this.options.logEndpoint);

        // Handler untuk event log
        this.eventSource.addEventListener('log', (event) => {
            const log = JSON.parse(event.data);
            this.addLogEntry(log);
        });

        // Handler untuk event ping (keep-alive)
        this.eventSource.addEventListener('ping', () => {
            // Silently handle ping events
        });

        // Handler untuk event timeout
        this.eventSource.addEventListener('timeout', (event) => {
            this.addLogEntry({
                level: 'warning',
                text: event.data,
                timestamp: new Date().toLocaleTimeString()
            });
            this.eventSource.close();
        });

        // Handler untuk error
        this.eventSource.onerror = () => {
            this.eventSource.close();
            this.addLogEntry({
                level: 'error',
                text: 'Koneksi ke stream log terputus',
                timestamp: new Date().toLocaleTimeString()
            });
        };
    }

    /**
     * Mulai polling status update secara berkala
     */
    startStatusPolling() {
        // Hentikan interval yang ada jika ada
        if (this.statusInterval) {
            clearInterval(this.statusInterval);
        }

        // Mulai interval baru
        this.statusInterval = setInterval(() => {
            fetch(this.options.statusEndpoint)
                .then(res => res.json())
                .then(data => this.updateProgressUI(data))
                .catch(err => {
                    console.error('Status polling error:', err);
                    // Jangan hentikan polling karena error, coba lagi
                });
        }, this.options.updateInterval);
    }

    /**
     * Update UI progress berdasarkan data status
     */
    updateProgressUI(data) {
        if (!data) return;

        // Update progress bar
        if (this.elements.progressValue && data.progress !== undefined) {
            this.elements.progressValue.style.width = data.progress + '%';
            this.elements.progressValue.innerText = data.progress + '%';

            // Ubah warna progress bar berdasarkan nilai
            if (data.progress >= 100) {
                this.elements.progressValue.classList.remove('bg-primary', 'bg-warning');
                this.elements.progressValue.classList.add('bg-success');
            } else if (data.progress === 0) {
                this.elements.progressValue.classList.remove('bg-primary', 'bg-success');
                this.elements.progressValue.classList.add('bg-warning');
            } else {
                this.elements.progressValue.classList.remove('bg-warning', 'bg-success');
                this.elements.progressValue.classList.add('bg-primary');
            }
        }

        // Update status text
        if (this.elements.statusBox && data.step) {
            this.elements.statusBox.innerText = data.step;
        }

        // Hentikan polling jika update selesai
        if (data.progress >= 100 || (data.progress === 0 && !this.isUpdating)) {
            clearInterval(this.statusInterval);
            this.statusInterval = null;
        }
    }

    /**
     * Hentikan monitoring (streaming dan polling)
     */
    stopMonitoring() {
        // Hentikan EventSource untuk log
        if (this.eventSource) {
            this.eventSource.close();
            this.eventSource = null;
        }

        // Hentikan interval polling status
        if (this.statusInterval) {
            clearInterval(this.statusInterval);
            this.statusInterval = null;
        }
    }

    /**
     * Set UI ke mode loading atau normal
     */
    setUILoading(loading) {
        // Set button loading state
        if (this.elements.startButton) {
            this.elements.startButton.disabled = loading;
        }

        // Toggle loading icon
        if (this.elements.loadingIcon) {
            this.elements.loadingIcon.style.display = loading ? 'inline-block' : 'none';
        }

        // Toggle cancel button visibility
        if (this.elements.cancelButton) {
            this.elements.cancelButton.classList.toggle('d-none', !loading);
        }
    }

    /**
     * Tambahkan pesan status ke status box
     */
    addStatusMessage(type, message) {
        if (!this.elements.statusBox) return;

        this.elements.statusBox.className = 'alert alert-' + type;
        this.elements.statusBox.classList.remove('d-none');
        this.elements.statusBox.innerText = message;
    }

    /**
     * Tambahkan entry log ke log console
     */
    addLogEntry(log) {
        if (!this.elements.logConsole) return;

        const logEntry = document.createElement('div');
        logEntry.className = `log-entry log-${log.level}`;

        const timestamp = document.createElement('span');
        timestamp.className = 'log-timestamp';
        timestamp.textContent = `[${log.timestamp}] `;

        const message = document.createElement('span');
        message.className = 'log-message';
        message.textContent = `${log.text}`;

        logEntry.appendChild(timestamp);
        logEntry.appendChild(message);

        if (log.details) {
            const details = document.createElement('div');
            details.className = 'log-details';
            details.textContent = typeof log.details === 'object'
                ? JSON.stringify(log.details, null, 2)
                : log.details;
            logEntry.appendChild(details);
        }

        this.elements.logConsole.appendChild(logEntry);
        this.elements.logConsole.scrollTop = this.elements.logConsole.scrollHeight;
    }
}

// Inisialisasi updater saat dokumen siap
document.addEventListener('DOMContentLoaded', function() {
    // Buat instance updater
    window.updater = new HeruUpdater();
});
