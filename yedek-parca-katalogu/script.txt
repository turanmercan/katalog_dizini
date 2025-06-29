document.addEventListener('DOMContentLoaded', function() {
    const catalogTree = document.getElementById('catalogTree');
    const vehicleNumberInput = document.getElementById('vehicleNumberInput');
    const searchByNumberBtn = document.getElementById('searchByNumberBtn');
    const searchMessage = document.getElementById('searchMessage');
    const welcomeSection = document.getElementById('welcomeSection');
    const pdfViewerContainer = document.querySelector('.pdf-viewer-container');
    const pdfFrame = document.getElementById('pdfFrame');

    let allCatalogsData = []; // Tüm katalog verilerini saklayacağımız dizi

    // data.json dosyasını yükle
    fetch('data.json')
        .then(response => {
            if (!response.ok) {
                throw new Error('data.json yüklenemedi: ' + response.statusText);
            }
            return response.json();
        })
        .then(data => {
            allCatalogsData = data; // Yüklenen veriyi sakla
            createCatalogTree(data, catalogTree); // Menüyü oluştur
        })
        .catch(error => {
            console.error('Katalog verisi yüklenirken bir hata oluştu:', error);
            catalogTree.innerHTML = '<li>Katalog verileri yüklenemedi. Lütfen konsolu kontrol edin.</li>';
            searchMessage.textContent = 'Katalog verileri yüklenemedi.';
            searchMessage.style.color = 'red';
        });

    // Menüyü (soy ağacını) dinamik olarak oluşturan fonksiyon
    function createCatalogTree(items, parentElement) {
        items.forEach(item => {
            const listItem = document.createElement('li');
            const link = document.createElement('a');

            // İkonu ekle
            if (item.icon) {
                const icon = document.createElement('i');
                icon.className = item.icon;
                link.appendChild(icon);
            }

            link.appendChild(document.createTextNode(item.name)); // Menü ismini ekle

            if (item.type === 'category' && item.children) {
                // Kategori ise, accordion için href="#" kullan
                link.href = '#';
                listItem.classList.add('has-submenu'); // CSS için sınıf ekle

                // Accordion olay dinleyicisi
                link.addEventListener('click', function(event) {
                    event.preventDefault(); // Varsayılan link davranışını engelle

                    const subMenu = listItem.querySelector('ul');
                    if (subMenu) {
                        // Diğer açık menüleri kapat
                        catalogTree.querySelectorAll('li.has-submenu > ul').forEach(function(ul) {
                            if (ul !== subMenu && ul.style.maxHeight !== '0px') {
                                ul.style.maxHeight = '0px';
                                ul.parentElement.classList.remove('active');
                            }
                        });

                        // Tıklanan menüyü aç/kapat
                        if (subMenu.style.maxHeight === '0px' || subMenu.style.maxHeight === '') {
                            subMenu.style.maxHeight = subMenu.scrollHeight + 'px';
                            listItem.classList.add('active');
                        } else {
                            subMenu.style.maxHeight = '0px';
                            listItem.classList.remove('active');
                        }
                    }
                });

                const subList = document.createElement('ul');
                subList.classList.add('submenu'); // Alt menü için sınıf
                listItem.appendChild(link);
                listItem.appendChild(subList);
                createCatalogTree(item.children, subList); // Rekürsif çağrı ile alt menüleri oluştur
            } else if (item.type === 'item' && item.pdf_path) {
                // PDF öğesi ise, PDF'i gösterme olayını ata
                link.href = '#'; // Sayfa atlamasını engelle
                link.dataset.pdfPath = item.pdf_path; // PDF yolunu veri özniteliği olarak sakla

                link.addEventListener('click', function(event) {
                    event.preventDefault(); // Sayfa yenilenmesini engelle
                    showPdf(this.dataset.pdfPath); // PDF'i gösterme fonksiyonunu çağır

                    // Aktif linki işaretle
                    catalogTree.querySelectorAll('li a').forEach(a => a.classList.remove('current-active-pdf'));
                    this.classList.add('current-active-pdf');
                });
                listItem.appendChild(link);
            }
            parentElement.appendChild(listItem);
        });
    }

    // PDF'i iframe içinde gösteren fonksiyon
    function showPdf(path) {
        welcomeSection.style.display = 'none'; // Hoş geldiniz mesajını gizle
        pdfViewerContainer.style.display = 'block'; // PDF görüntüleyiciyi göster
        pdfFrame.src = path; // PDF'i yükle
    }

    // Araç Numarası ile Arama Fonksiyonu
    searchByNumberBtn.addEventListener('click', performSearch);
    vehicleNumberInput.addEventListener('keydown', function(event) {
        if (event.key === 'Enter') {
            performSearch();
        }
    });

    function performSearch() {
        const searchTerm = vehicleNumberInput.value.trim().toUpperCase(); // Girişi al, boşlukları temizle, büyük harfe çevir
        searchMessage.textContent = ''; // Önceki mesajı temizle
        searchMessage.style.color = 'initial';

        if (searchTerm === '') {
            searchMessage.textContent = 'Lütfen bir araç numarası girin.';
            searchMessage.style.color = 'orange';
            return;
        }

        let foundItem = null;
        // Tüm katalog verisi içinde ara (rekürsif arama)
        function searchInItems(items) {
            for (const item of items) {
                if (item.type === 'item') {
                    if (item.vehicle_numbers && item.vehicle_numbers.some(num => num.toUpperCase().includes(searchTerm))) {
                        foundItem = item;
                        return true; // Bulundu
                    }
                } else if (item.type === 'category' && item.children) {
                    if (searchInItems(item.children)) { // Alt kategorilerde ara
                        return true;
                    }
                }
            }
            return false;
        }

        searchInItems(allCatalogsData); // Tüm ana veri setinde aramayı başlat

        if (foundItem) {
            searchMessage.textContent = `Bulundu: ${foundItem.name}`;
            searchMessage.style.color = 'green';
            showPdf(foundItem.pdf_path); // Bulunan PDF'i göster

            // Menüde ilgili öğeyi işaretle ve aç (isteğe bağlı ama kullanıcı deneyimi için iyi)
            // Bu kısım biraz daha karmaşık olabilir, şimdilik sadece PDF'i açalım.
            // İleri aşamada menüde ilgili öğeyi genişletebiliriz.
            
            // Tüm linklerdeki aktif sınıfı kaldır
            catalogTree.querySelectorAll('li a').forEach(a => a.classList.remove('current-active-pdf'));
            // İlgili linki bulup işaretle (name'e göre)
            const correspondingLink = Array.from(catalogTree.querySelectorAll('li a')).find(link => link.textContent.includes(foundItem.name));
            if (correspondingLink) {
                correspondingLink.classList.add('current-active-pdf');
                // Eğer üst kategorisi kapalıysa onu da aç
                let parentUl = correspondingLink.closest('ul');
                while(parentUl && parentUl.classList.contains('submenu')) {
                    parentUl.style.maxHeight = parentUl.scrollHeight + 'px';
                    parentUl.parentElement.classList.add('active');
                    parentUl = parentUl.parentElement.closest('ul');
                }
            }

        } else {
            searchMessage.textContent = `"${searchTerm}" numarasına sahip araç bulunamadı.`;
            searchMessage.style.color = 'red';
        }
    }
});