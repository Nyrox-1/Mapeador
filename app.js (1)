// ==========================================
// TRANSLATION SYSTEM
// ==========================================

const translations = {
    en: {
        app_title: "Pack stack generator",
        upload_title: "Upload addon files",
        upload_description: "Drag & drop .json or .zip files here or click to browse",
        browse_button: "Browse files",
        packs_title: "Loaded packs",
        clear_all: "Clear all",
        manifest_preview_title: "Manifest preview",
        behavior_packs: "Behavior packs",
        resource_packs: "Resource packs",
        output_title: "Generated JSON files",
        copy: "Copy",
        download: "Download",
        dependencies: "Dependencies",
        uuid: "UUID",
        version: "Version",
        error_invalid_json: "Invalid JSON",
        error_no_manifest: "No valid manifest.json found",
        error_missing_uuid: "Missing header.uuid",
        error_invalid_type: "Invalid pack type",
        error_missing_dependency: "Missing dependency",
        success_file_loaded: "File loaded successfully",
        success_packs_loaded: "packs loaded",
        success_copied: "Copied to clipboard",
        success_downloaded: "File downloaded",
        warning_duplicate: "Duplicate pack (same UUID)"
    },
    es: {
        app_title: "Addon stack generator",
        upload_title: "Subir archivos de addons",
        upload_description: "Arrastra y suelta archivos .json o .zip aquí o haz clic para explorar",
        browse_button: "Explorar archivos",
        packs_title: "Packs cargados",
        clear_all: "Limpiar todo",
        manifest_preview_title: "Vista previa de manifiest",
        behavior_packs: "Packs de comportamiento",
        resource_packs: "Packs de recursos",
        output_title: "Archivos JSON generados",
        copy: "Copiar",
        download: "Descargar",
        dependencies: "Dependencias",
        uuid: "UUID",
        version: "Versión",
        error_invalid_json: "JSON inválido",
        error_no_manifest: "No se encontró manifest.json válido",
        error_missing_uuid: "Falta header.uuid",
        error_invalid_type: "Tipo de pack inválido",
        error_missing_dependency: "Dependencia faltante",
        success_file_loaded: "Archivo cargado exitosamente",
        success_packs_loaded: "packs cargados",
        success_copied: "Copiado al portapapeles",
        success_downloaded: "Archivo descargado",
        warning_duplicate: "Pack duplicado (mismo UUID)"
    }
};

let currentLang = 'en';

function translatePage() {
    document.querySelectorAll('[data-i18n]').forEach(element => {
        const key = element.getAttribute('data-i18n');
        if (translations[currentLang][key]) {
            element.textContent = translations[currentLang][key];
        }
    });
}

function t(key) {
    return translations[currentLang][key] || key;
}

// ==========================================
// STATE MANAGEMENT
// ==========================================

let packs = [];
let draggedIndex = null;

// ==========================================
// PACK VALIDATION & PROCESSING
// ==========================================

function isValidManifest(manifest) {
    // Check if it has the required structure for a Bedrock manifest
    if (!manifest.header || !manifest.modules) {
        return false;
    }
    
    if (!manifest.header.uuid || !manifest.header.version) {
        return false;
    }
    
    // Check if modules is an array and has at least one module
    if (!Array.isArray(manifest.modules) || manifest.modules.length === 0) {
        return false;
    }
    
    return true;
}

function detectPackType(manifest) {
    // Detect pack type based on modules
    if (!manifest.modules || !Array.isArray(manifest.modules)) {
        return null;
    }
    
    for (const module of manifest.modules) {
        if (module.type === 'data') {
            return 'BP'; // Behavior Pack
        } else if (module.type === 'resources') {
            return 'RP'; // Resource Pack
        }
    }
    
    return null;
}

function extractDependencies(manifest) {
    if (!manifest.dependencies || !Array.isArray(manifest.dependencies)) {
        return [];
    }
    
    return manifest.dependencies.map(dep => ({
        uuid: dep.uuid,
        version: dep.version
    }));
}

function checkDependencies(pack) {
    if (!pack.dependencies || pack.dependencies.length === 0) {
        return [];
    }
    
    return pack.dependencies.map(dep => {
        const found = packs.find(p => p.uuid === dep.uuid);
        return {
            ...dep,
            satisfied: !!found
        };
    });
}

// ==========================================
// FILE PROCESSING - IMPROVED WITH RECURSIVE SEARCH
// ==========================================

async function processJSONFile(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = (e) => {
            try {
                const manifest = JSON.parse(e.target.result);
                
                if (!isValidManifest(manifest)) {
                    reject({ error: 'error_invalid_json', file: file.name });
                    return;
                }
                
                const packType = detectPackType(manifest);
                if (!packType) {
                    reject({ error: 'error_invalid_type', file: file.name });
                    return;
                }
                
                const pack = {
                    id: Date.now() + Math.random(),
                    name: manifest.header.name || file.name,
                    uuid: manifest.header.uuid,
                    version: manifest.header.version,
                    type: packType,
                    dependencies: extractDependencies(manifest),
                    manifest: manifest,
                    fileName: file.name
                };
                
                resolve([pack]); // Return as array for consistency
            } catch (err) {
                reject({ error: 'error_invalid_json', file: file.name });
            }
        };
        
        reader.onerror = () => reject({ error: 'error_invalid_json', file: file.name });
        reader.readAsText(file);
    });
}

// IMPROVED: Recursive search for ALL manifest.json files in ALL folders
async function findAllManifestsInZip(zip) {
    const manifests = [];
    
    // Iterate through ALL files in the ZIP recursively
    for (const [path, zipEntry] of Object.entries(zip.files)) {
        // Check if the file is named manifest.json (case-insensitive)
        if (zipEntry.name.toLowerCase().endsWith('manifest.json') && !zipEntry.dir) {
            try {
                const content = await zipEntry.async('string');
                const manifest = JSON.parse(content);
                
                // Validate the manifest
                if (isValidManifest(manifest)) {
                    const packType = detectPackType(manifest);
                    if (packType) {
                        manifests.push({
                            manifest: manifest,
                            path: path,
                            packType: packType
                        });
                    }
                }
            } catch (err) {
                // Invalid JSON or parsing error, skip this file
                console.warn(`Skipping invalid manifest at ${path}:`, err);
            }
        }
    }
    
    return manifests;
}

// IMPROVED: Process ZIP files to find ALL manifests
async function processZIPFile(file) {
    return new Promise(async (resolve, reject) => {
        try {
            const zip = await JSZip.loadAsync(file);
            
            // Find ALL manifests recursively
            const foundManifests = await findAllManifestsInZip(zip);
            
            if (foundManifests.length === 0) {
                reject({ error: 'error_no_manifest', file: file.name });
                return;
            }
            
            // Create pack objects for all found manifests
            const packs = foundManifests.map((manifestData, index) => ({
                id: Date.now() + Math.random() + index,
                name: manifestData.manifest.header.name || `${file.name} (${manifestData.packType})`,
                uuid: manifestData.manifest.header.uuid,
                version: manifestData.manifest.header.version,
                type: manifestData.packType,
                dependencies: extractDependencies(manifestData.manifest),
                manifest: manifestData.manifest,
                fileName: file.name,
                manifestPath: manifestData.path
            }));
            
            resolve(packs); // Return array of packs
        } catch (err) {
            reject({ error: 'error_invalid_json', file: file.name });
        }
    });
}

// IMPROVED: Process any file type
async function processFile(file) {
    const fileName = file.name.toLowerCase();
    
    // Try to process as JSON if it has .json extension
    if (fileName.endsWith('.json')) {
        return await processJSONFile(file);
    }
    
    // Try to process as ZIP for .zip, .mcpack, .mcaddon, or any other file
    try {
        // Attempt to read as ZIP regardless of extension
        const packs = await processZIPFile(file);
        return packs;
    } catch (zipError) {
        // If it's not a valid ZIP and not a JSON, try reading as JSON anyway
        if (!fileName.endsWith('.json')) {
            try {
                return await processJSONFile(file);
            } catch (jsonError) {
                // Neither ZIP nor JSON worked
                throw { error: 'error_no_manifest', file: file.name };
            }
        }
        throw zipError;
    }
}

// ==========================================
// PACK MANAGEMENT
// ==========================================

function addPack(pack) {
    // Check for duplicates by UUID
    const exists = packs.find(p => p.uuid === pack.uuid);
    if (exists) {
        showToast('warning', `${t('warning_duplicate')}: ${pack.name}`);
        return false;
    }
    
    packs.push(pack);
    return true;
}

function removePack(index) {
    packs.splice(index, 1);
    updateUI();
}

function clearAllPacks() {
    packs = [];
    updateUI();
}

function reorderPacks(fromIndex, toIndex) {
    const [movedPack] = packs.splice(fromIndex, 1);
    packs.splice(toIndex, 0, movedPack);
    updateUI();
}

// ==========================================
// UI RENDERING
// ==========================================

function renderPackCard(pack, index) {
    const deps = checkDependencies(pack);
    const typeClass = pack.type.toLowerCase();
    const typeLabel = pack.type === 'BP' ? 'BP' : 'RP';
    
    return `
        <div class="pack-card" draggable="true" data-index="${index}">
            <div class="pack-header">
                <span class="pack-type ${typeClass}">
                    <svg class="pack-type-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        ${pack.type === 'BP' ? 
                            '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10"></path>' :
                            '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z"></path>'
                        }
                    </svg>
                    ${typeLabel}
                </span>
                <button class="pack-remove" onclick="removePack(${index})">
                    <svg fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
                    </svg>
                </button>
            </div>
            <div class="pack-name">${pack.name}</div>
            <div class="pack-info">
                <div class="pack-info-item">
                    <svg class="pack-info-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 7h.01M7 3h5c.512 0 1.024.195 1.414.586l7 7a2 2 0 010 2.828l-7 7a2 2 0 01-2.828 0l-7-7A1.994 1.994 0 013 12V7a4 4 0 014-4z"></path>
                    </svg>
                    <span class="pack-info-label">${t('uuid')}:</span>
                    <span class="pack-info-value">${pack.uuid}</span>
                </div>
                <div class="pack-info-item">
                    <svg class="pack-info-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 20l4-16m2 16l4-16M6 9h14M4 15h14"></path>
                    </svg>
                    <span class="pack-info-label">${t('version')}:</span>
                    <span class="pack-info-value">${Array.isArray(pack.version) ? pack.version.join('.') : pack.version}</span>
                </div>
                ${pack.manifestPath ? `
                <div class="pack-info-item">
                    <svg class="pack-info-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 7v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-6l-2-2H5a2 2 0 00-2 2z"></path>
                    </svg>
                    <span class="pack-info-label">Path:</span>
                    <span class="pack-info-value" style="font-size: 0.75rem; word-break: break-all;">${pack.manifestPath}</span>
                </div>
                ` : ''}
            </div>
            ${deps.length > 0 ? `
                <div class="pack-dependencies">
                    <div class="pack-dependencies-title">${t('dependencies')}:</div>
                    ${deps.map(dep => `
                        <div class="dependency-item">
                            <span class="dependency-status ${dep.satisfied ? 'satisfied' : 'missing'}"></span>
                            <span>${dep.uuid}</span>
                        </div>
                    `).join('')}
                </div>
            ` : ''}
        </div>
    `;
}

function renderManifestPreview() {
    const bpList = document.getElementById('bpManifestList');
    const rpList = document.getElementById('rpManifestList');
    
    const bpPacks = packs.filter(p => p.type === 'BP');
    const rpPacks = packs.filter(p => p.type === 'RP');
    
    if (bpPacks.length === 0) {
        bpList.innerHTML = '<div class="empty-state"><p>No Behavior Packs loaded</p></div>';
    } else {
        bpList.innerHTML = bpPacks.map(pack => `
            <div class="manifest-item">
                <div class="manifest-item-label">${t('uuid')}</div>
                <div class="manifest-item-value">${pack.uuid}</div>
                <div class="manifest-item-label" style="margin-top: 0.5rem;">${t('version')}</div>
                <div class="manifest-item-value">${Array.isArray(pack.version) ? pack.version.join('.') : pack.version}</div>
            </div>
        `).join('');
    }
    
    if (rpPacks.length === 0) {
        rpList.innerHTML = '<div class="empty-state"><p>No Resource Packs loaded</p></div>';
    } else {
        rpList.innerHTML = rpPacks.map(pack => `
            <div class="manifest-item">
                <div class="manifest-item-label">${t('uuid')}</div>
                <div class="manifest-item-value">${pack.uuid}</div>
                <div class="manifest-item-label" style="margin-top: 0.5rem;">${t('version')}</div>
                <div class="manifest-item-value">${Array.isArray(pack.version) ? pack.version.join('.') : pack.version}</div>
            </div>
        `).join('');
    }
}

function renderOutputJSON() {
    const bpPacks = packs.filter(p => p.type === 'BP');
    const rpPacks = packs.filter(p => p.type === 'RP');
    
    const bpOutputCard = document.getElementById('bpOutputCard');
    const rpOutputCard = document.getElementById('rpOutputCard');
    const bpOutput = document.getElementById('bpOutput');
    const rpOutput = document.getElementById('rpOutput');
    
    // Generate Behavior Packs JSON
    if (bpPacks.length > 0) {
        const bpJSON = bpPacks.map(pack => ({
            pack_id: pack.uuid,
            version: pack.version
        }));
        bpOutput.textContent = JSON.stringify(bpJSON, null, 2);
        bpOutputCard.style.display = 'block';
    } else {
        bpOutputCard.style.display = 'none';
    }
    
    // Generate Resource Packs JSON
    if (rpPacks.length > 0) {
        const rpJSON = rpPacks.map(pack => ({
            pack_id: pack.uuid,
            version: pack.version
        }));
        rpOutput.textContent = JSON.stringify(rpJSON, null, 2);
        rpOutputCard.style.display = 'block';
    } else {
        rpOutputCard.style.display = 'none';
    }
}

function updateUI() {
    const packsSection = document.getElementById('packsSection');
    const manifestPreviewSection = document.getElementById('manifestPreviewSection');
    const outputSection = document.getElementById('outputSection');
    const packsGrid = document.getElementById('packsGrid');
    
    if (packs.length === 0) {
        packsSection.style.display = 'none';
        manifestPreviewSection.style.display = 'none';
        outputSection.style.display = 'none';
        return;
    }
    
    packsSection.style.display = 'block';
    manifestPreviewSection.style.display = 'block';
    outputSection.style.display = 'block';
    
    packsGrid.innerHTML = packs.map((pack, index) => renderPackCard(pack, index)).join('');
    
    // Re-attach drag and drop listeners
    attachDragListeners();
    
    renderManifestPreview();
    renderOutputJSON();
}

// ==========================================
// DRAG AND DROP FOR REORDERING
// ==========================================

function attachDragListeners() {
    const packCards = document.querySelectorAll('.pack-card');
    
    packCards.forEach((card, index) => {
        card.addEventListener('dragstart', handleDragStart);
        card.addEventListener('dragover', handleDragOver);
        card.addEventListener('drop', handleDrop);
        card.addEventListener('dragend', handleDragEnd);
    });
}

function handleDragStart(e) {
    draggedIndex = parseInt(this.getAttribute('data-index'));
    this.classList.add('dragging');
}

function handleDragOver(e) {
    e.preventDefault();
    return false;
}

function handleDrop(e) {
    e.preventDefault();
    const dropIndex = parseInt(this.getAttribute('data-index'));
    
    if (draggedIndex !== null && draggedIndex !== dropIndex) {
        reorderPacks(draggedIndex, dropIndex);
    }
    
    return false;
}

function handleDragEnd(e) {
    this.classList.remove('dragging');
    draggedIndex = null;
}

// ==========================================
// FILE UPLOAD HANDLING - IMPROVED
// ==========================================

async function handleFiles(files) {
    const fileArray = Array.from(files);
    let totalPacksAdded = 0;
    let errors = 0;
    
    for (const file of fileArray) {
        try {
            const packsFromFile = await processFile(file);
            
            // Add all packs found in the file
            let addedFromFile = 0;
            for (const pack of packsFromFile) {
                const added = addPack(pack);
                if (added) {
                    addedFromFile++;
                    totalPacksAdded++;
                }
            }
            
            if (addedFromFile > 0) {
                if (addedFromFile === 1) {
                    showToast('success', `${t('success_file_loaded')}: ${file.name}`);
                } else {
                    showToast('success', `${addedFromFile} ${t('success_packs_loaded')}: ${file.name}`);
                }
            }
        } catch (err) {
            showError(err.error, err.file);
            errors++;
        }
    }
    
    updateUI();
    
    if (totalPacksAdded > 1 && fileArray.length > 1) {
        showToast('success', `Total: ${totalPacksAdded} ${t('success_packs_loaded')}`);
    }
}

// ==========================================
// COPY & DOWNLOAD FUNCTIONS
// ==========================================

async function copyToClipboard(text) {
    try {
        await navigator.clipboard.writeText(text);
        showToast('success', t('success_copied'));
    } catch (err) {
        showToast('error', 'Failed to copy');
    }
}

function downloadJSON(filename, content) {
    const blob = new Blob([content], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
    showToast('success', t('success_downloaded') + `: ${filename}`);
}

// ==========================================
// ERROR & TOAST NOTIFICATIONS
// ==========================================

function showError(errorKey, details = '') {
    const errorContainer = document.getElementById('errorContainer');
    
    const errorEl = document.createElement('div');
    errorEl.className = 'error-message';
    errorEl.innerHTML = `
        <svg class="error-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
        </svg>
        <div class="error-content">
            <div class="error-title">${t(errorKey)}</div>
            ${details ? `<div class="error-description">${details}</div>` : ''}
        </div>
        <button class="error-close">
            <svg fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
            </svg>
        </button>
    `;
    
    errorContainer.appendChild(errorEl);
    
    errorEl.querySelector('.error-close').addEventListener('click', () => {
        errorEl.remove();
    });
    
    setTimeout(() => {
        errorEl.remove();
    }, 5000);
}

function showToast(type, message) {
    const toastContainer = document.getElementById('toastContainer');
    
    const toast = document.createElement('div');
    toast.className = `toast ${type}`;
    toast.innerHTML = `
        <svg class="toast-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            ${type === 'success' ? 
                '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"></path>' :
                type === 'warning' ?
                '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"></path>' :
                '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>'
            }
        </svg>
        <div class="toast-message">${message}</div>
    `;
    
    toastContainer.appendChild(toast);
    
    setTimeout(() => {
        toast.remove();
    }, 3000);
}

// ==========================================
// EVENT LISTENERS
// ==========================================

document.addEventListener('DOMContentLoaded', () => {
    // Language selector
    const languageSelector = document.getElementById('languageSelector');
    languageSelector.addEventListener('change', (e) => {
        currentLang = e.target.value;
        translatePage();
        updateUI(); // Re-render to apply translations
    });
    
    // Upload area
    const uploadArea = document.getElementById('uploadArea');
    const fileInput = document.getElementById('fileInput');
    const browseBtn = document.getElementById('browseBtn');
    
    uploadArea.addEventListener('click', () => fileInput.click());
    browseBtn.addEventListener('click', (e) => {
        e.stopPropagation();
        fileInput.click();
    });
    
    fileInput.addEventListener('change', (e) => {
        handleFiles(e.target.files);
        e.target.value = ''; // Reset input
    });
    
    // Drag and drop
    uploadArea.addEventListener('dragover', (e) => {
        e.preventDefault();
        uploadArea.classList.add('drag-over');
    });
    
    uploadArea.addEventListener('dragleave', () => {
        uploadArea.classList.remove('drag-over');
    });
    
    uploadArea.addEventListener('drop', (e) => {
        e.preventDefault();
        uploadArea.classList.remove('drag-over');
        handleFiles(e.dataTransfer.files);
    });
    
    // Clear all button
    document.getElementById('clearAllBtn').addEventListener('click', clearAllPacks);
    
    // Copy and download buttons
    document.getElementById('copyBPBtn').addEventListener('click', () => {
        const content = document.getElementById('bpOutput').textContent;
        copyToClipboard(content);
    });
    
    document.getElementById('downloadBPBtn').addEventListener('click', () => {
        const content = document.getElementById('bpOutput').textContent;
        downloadJSON('world_behavior_packs.json', content);
    });
    
    document.getElementById('copyRPBtn').addEventListener('click', () => {
        const content = document.getElementById('rpOutput').textContent;
        copyToClipboard(content);
    });
    
    document.getElementById('downloadRPBtn').addEventListener('click', () => {
        const content = document.getElementById('rpOutput').textContent;
        downloadJSON('world_resource_packs.json', content);
    });
    
    // Initial translation
    translatePage();
});
