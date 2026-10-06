const STORAGE_KEY = 'libraryBooks';
const THEME_KEY = 'libraryTheme';
const PAGE_THICKNESS_MM = 0.055;
const MIN_THICKNESS_MM = 8;
const MAX_THICKNESS_MM = 70;
const SHELF_COUNT = 3;
const SHELF_GAP = 8;
const SHELF_WIDTH = 980;

const elements = {
  searchInput: document.getElementById('searchInput'),
  genreFilter: document.getElementById('genreFilter'),
  addBookBtn: document.getElementById('addBookBtn'),
  bookshelfContainer: document.getElementById('bookshelfContainer'),
  searchStatus: document.getElementById('searchStatus'),
  prevShelfBtn: document.getElementById('prevShelfBtn'),
  nextShelfBtn: document.getElementById('nextShelfBtn'),
  bookModal: document.getElementById('bookModal'),
  bookModalTitle: document.getElementById('bookModalTitle'),
  bookForm: document.getElementById('bookForm'),
  formErrors: document.getElementById('formErrors'),
  themeToggle: document.getElementById('themeToggle'),
  libraryDropdown: document.getElementById('libraryDropdown'),
  libraryMenuButton: document.getElementById('libraryMenuButton'),
  exportLibraryBtn: document.getElementById('exportLibraryBtn'),
  importLibraryBtn: document.getElementById('importLibraryBtn'),
  clearSampleBooksBtn: document.getElementById('clearSampleBooksBtn'),
  importFileInput: document.getElementById('importFileInput'),
  confirmModal: document.getElementById('confirmModal'),
  confirmDeleteBtn: document.getElementById('confirmDeleteBtn'),
  cancelDeleteBtn: document.getElementById('cancelDeleteBtn'),
  bookDetailModal: document.getElementById('bookDetailModal'),
  bookDetailContent: document.getElementById('bookDetailContent'),
  toastContainer: document.getElementById('toastContainer'),
  leafLayer: document.getElementById('leafLayer'),
  bookId: document.getElementById('bookId'),
  bookTitle: document.getElementById('bookTitle'),
  bookAuthor: document.getElementById('bookAuthor'),
  bookGenre: document.getElementById('bookGenre'),
  bookDescription: document.getElementById('bookDescription'),
  bookPages: document.getElementById('bookPages'),
  bookHeight: document.getElementById('bookHeight'),
  bookWidth: document.getElementById('bookWidth'),
  bookThickness: document.getElementById('bookThickness'),
  dimensionUnit: document.getElementById('dimensionUnit'),
  bookProgress: document.getElementById('bookProgress'),
  bookRating: document.getElementById('bookRating'),
  bookColor: document.getElementById('bookColor'),
  colorLabel: document.getElementById('colorLabel'),
  bookShelfPosition: document.getElementById('bookShelfPosition'),
  coverImageInput: document.getElementById('coverImageInput'),
  spineImageInput: document.getElementById('spineImageInput'),
  coverPreview: document.getElementById('coverPreview'),
  spinePreview: document.getElementById('spinePreview'),
  genreSuggestions: document.getElementById('genreSuggestions'),
};

const state = {
  books: [],
  currentPageIndex: 0,
  searchTerm: '',
  activeGenre: 'All Genres',
  selectedBookId: null,
  editingBookId: null,
  currentTheme: 'light',
};

const SAMPLE_GENRES = [
  'Fantasy',
  'Romance',
  'Mystery',
  'Thriller',
  'Science Fiction',
  'History',
  'Psychology',
  'Business',
  'Economics',
  'Self-help',
  'Philosophy',
  'Biography',
  'Technology',
  'Fiction',
  'Dystopian',
  'Other'
];

document.addEventListener('DOMContentLoaded', initApp);

function initApp() {
  bindEvents();
  state.currentTheme = localStorage.getItem(THEME_KEY) || 'light';
  applyTheme();
  createAutumnLeaves();
  loadFromLocalStorage();
  renderApp();
}

function bindEvents() {
  elements.searchInput.addEventListener('input', (event) => {
    state.searchTerm = event.target.value.trim();
    renderApp();
  });

  elements.genreFilter.addEventListener('change', (event) => {
    state.activeGenre = event.target.value;
    state.currentPageIndex = 0;
    renderApp();
  });

  elements.addBookBtn.addEventListener('click', () => openAddBookModal());

  elements.themeToggle.addEventListener('click', () => {
    state.currentTheme = state.currentTheme === 'light' ? 'night' : 'light';
    localStorage.setItem(THEME_KEY, state.currentTheme);
    applyTheme();
  });

  elements.libraryMenuButton.addEventListener('click', () => {
    const isOpen = !elements.libraryDropdown.classList.contains('hidden');
    elements.libraryDropdown.classList.toggle('hidden', isOpen);
    elements.libraryMenuButton.setAttribute('aria-expanded', String(!isOpen));
  });

  document.addEventListener('click', (event) => {
    const clickedInsideMenu = event.target.closest('.menu-wrap');
    if (!clickedInsideMenu) {
      elements.libraryDropdown.classList.add('hidden');
      elements.libraryMenuButton.setAttribute('aria-expanded', 'false');
    }

    if (event.target.matches('[data-close-modal="true"]')) {
      closeModals();
    }
  });

  elements.exportLibraryBtn.addEventListener('click', exportLibrary);
  elements.importLibraryBtn.addEventListener('click', () => elements.importFileInput.click());
  elements.importFileInput.addEventListener('change', importLibraryFromFile);
  elements.clearSampleBooksBtn.addEventListener('click', clearSampleBooks);

  elements.prevShelfBtn.addEventListener('click', () => {
    if (state.currentPageIndex > 0) {
      state.currentPageIndex -= 1;
      renderBookshelf();
    }
  });

  elements.nextShelfBtn.addEventListener('click', () => {
    const pages = calculateShelfLayout(filterBooks());
    if (state.currentPageIndex < pages.length - 1) {
      state.currentPageIndex += 1;
      renderBookshelf();
    }
  });

  elements.bookForm.addEventListener('submit', handleBookFormSubmit);
  elements.coverImageInput.addEventListener('change', handleImageUpload.bind(null, 'cover'));
  elements.spineImageInput.addEventListener('change', handleImageUpload.bind(null, 'spine'));
  elements.cancelDeleteBtn.addEventListener('click', closeDeleteModal);
  elements.confirmDeleteBtn.addEventListener('click', confirmDeleteBook);
  elements.bookColor.addEventListener('input', () => {
    elements.colorLabel.textContent = elements.bookColor.value;
  });

  document.addEventListener('keydown', (event) => {
    if (event.key === 'Escape') {
      closeModals();
    }
  });

  document.querySelectorAll('[data-close-modal="true"]').forEach((button) => {
    button.addEventListener('click', closeModals);
  });
}

function applyTheme() {
  document.body.setAttribute('data-theme', state.currentTheme === 'light' ? 'light' : 'night');
  elements.themeToggle.textContent = state.currentTheme === 'light' ? 'Autumn Night' : 'Autumn Light';
  elements.themeToggle.setAttribute('aria-label', `Switch to ${state.currentTheme === 'light' ? 'light' : 'night'} theme`);
}

function renderApp() {
  renderFilterOptions();
  renderBookshelf();
}

function renderFilterOptions() {
  const uniqueGenres = Array.from(new Set(state.books.map((book) => book.genre).filter(Boolean))).sort((a, b) => a.localeCompare(b));
  const options = ['All Genres', ...uniqueGenres];

  const currentValue = options.includes(state.activeGenre) ? state.activeGenre : 'All Genres';
  state.activeGenre = currentValue;

  elements.genreFilter.innerHTML = options
    .map((genre) => `<option value="${escapeHtml(genre)}">${escapeHtml(genre)}</option>`)
    .join('');
  elements.genreFilter.value = currentValue;

  const customGenres = [...SAMPLE_GENRES, ...uniqueGenres];
  elements.genreSuggestions.innerHTML = Array.from(new Set(customGenres))
    .map((genre) => `<option value="${escapeHtml(genre)}"></option>`)
    .join('');
}

function renderBookshelf() {
  const visibleBooks = filterBooks();
  const pages = calculateShelfLayout(visibleBooks);

  if (pages.length === 0) {
    state.currentPageIndex = 0;
    renderEmptyLibraryState();
    return;
  }

  if (state.currentPageIndex >= pages.length) {
    state.currentPageIndex = pages.length - 1;
  }

  elements.bookshelfContainer.innerHTML = '';

  const currentPage = pages[state.currentPageIndex];
  const pageContainer = document.createElement('div');
  pageContainer.className = 'bookshelf-page';

  const frame = document.createElement('div');
  frame.className = 'bookshelf-frame';

  currentPage.shelves.forEach((shelfBooks) => {
    const shelf = document.createElement('div');
    shelf.className = 'shelf';
    shelf.setAttribute('aria-label', 'Book shelf');

    const stack = document.createElement('div');
    stack.className = 'books';

    shelfBooks.forEach((book) => {
      const bookEl = createBookElement(book);
      stack.appendChild(bookEl);
    });

    shelf.appendChild(stack);
    frame.appendChild(shelf);
  });

  pageContainer.appendChild(frame);
  elements.bookshelfContainer.appendChild(pageContainer);

  const totalPages = pages.length;
  elements.prevShelfBtn.disabled = state.currentPageIndex === 0;
  elements.nextShelfBtn.disabled = state.currentPageIndex >= totalPages - 1;

  const statusText = totalPages > 1
    ? `Bookshelf ${state.currentPageIndex + 1} of ${totalPages}`
    : 'Shelf 1 / 1';

  elements.searchStatus.textContent = `${visibleBooks.length} book${visibleBooks.length === 1 ? '' : 's'} found • ${statusText}`;

  if (visibleBooks.length === 0) {
    elements.searchStatus.textContent = 'No books found.';
  }
}

function renderEmptyLibraryState() {
  elements.bookshelfContainer.innerHTML = `
    <div class="bookshelf-page">
      <div class="bookshelf-frame">
        <div class="empty-library-state">
          <h2>Your shelves are waiting.</h2>
          <p>Add your first book and begin building your library.</p>
          <button class="primary-button" type="button" data-trigger-add-book="true">ADD YOUR FIRST BOOK</button>
        </div>
      </div>
    </div>
  `;

  const addButton = elements.bookshelfContainer.querySelector('[data-trigger-add-book="true"]');
  if (addButton) {
    addButton.addEventListener('click', openAddBookModal);
  }

  elements.prevShelfBtn.disabled = true;
  elements.nextShelfBtn.disabled = true;
  elements.searchStatus.textContent = 'No books found.';
}

function filterBooks() {
  const query = state.searchTerm.toLowerCase();

  return state.books.filter((book) => {
    const matchesSearch = !query || `${book.title} ${book.author}`.toLowerCase().includes(query);
    const matchesGenre = state.activeGenre === 'All Genres' || book.genre === state.activeGenre;
    return matchesSearch && matchesGenre;
  });
}

function calculateShelfLayout(booksToPlace) {
  const pages = [];
  const pageWidth = SHELF_WIDTH;

  if (!booksToPlace.length) {
    return pages;
  }

  let currentPage = { shelves: Array.from({ length: SHELF_COUNT }, () => []) };

  booksToPlace.forEach((book) => {
    const visualWidth = calculateBookVisualWidth(book);
    let placed = false;

    for (let shelfIndex = 0; shelfIndex < SHELF_COUNT; shelfIndex += 1) {
      const shelf = currentPage.shelves[shelfIndex];
      const currentUsedWidth = shelf.reduce((sum, item) => sum + item.visualWidth + SHELF_GAP, 0);

      if (currentUsedWidth + visualWidth + SHELF_GAP <= pageWidth) {
        const placement = {
          ...book,
          visualWidth,
          rotation: generateBookRotation(shelf.length, shelfIndex),
          translateX: generateBookOffset(shelf.length),
        };

        shelf.push(placement);
        placed = true;
        break;
      }
    }

    if (!placed) {
      pages.push(currentPage);
      currentPage = { shelves: Array.from({ length: SHELF_COUNT }, () => []) };

      for (let shelfIndex = 0; shelfIndex < SHELF_COUNT; shelfIndex += 1) {
        const shelf = currentPage.shelves[shelfIndex];
        const currentUsedWidth = shelf.reduce((sum, item) => sum + item.visualWidth + SHELF_GAP, 0);

        if (currentUsedWidth + visualWidth + SHELF_GAP <= pageWidth) {
          shelf.push({ ...book, visualWidth, rotation: generateBookRotation(shelf.length, shelfIndex), translateX: generateBookOffset(shelf.length) });
          placed = true;
          break;
        }
      }

      if (!placed) {
        currentPage = { shelves: Array.from({ length: SHELF_COUNT }, () => []) };
        currentPage.shelves[0].push({ ...book, visualWidth, rotation: generateBookRotation(0, 0), translateX: generateBookOffset(0) });
      }
    }
  });

  pages.push(currentPage);
  return pages;
}

function createBookElement(book) {
  const bookEl = document.createElement('button');
  bookEl.type = 'button';
  bookEl.className = 'book';
  bookEl.dataset.bookId = book.id;
  bookEl.style.setProperty('--book-width', `${book.visualWidth || calculateBookVisualWidth(book)}px`);
  bookEl.style.setProperty('--book-height', `${calculateBookVisualHeight(book)}px`);
  bookEl.style.setProperty('--book-color', book.color || '#7a4432');
  bookEl.style.setProperty('--book-rotation', `${book.rotation || 0}deg`);
  bookEl.style.setProperty('--book-translate-x', `${book.translateX || 0}px`);
  bookEl.style.setProperty('--book-shadow', `${book.shadow || 'rgba(30,18,14,0.18)'}`);

  const spineStyle = book.spineImage ? `background-image: url('${book.spineImage}'); background-size: cover; background-position: center;` : '';

  const label = escapeHtml(book.title || 'Untitled');

  if (book.spineImage) {
    bookEl.innerHTML = `
      <img class="book-spine-image" src="${book.spineImage}" alt="" aria-hidden="true" />
      <span class="book-label" data-text="${label}">${truncateTitle(label)}</span>
    `;
  } else {
    bookEl.innerHTML = `
      <span class="book-spine-pattern" aria-hidden="true" style="${spineStyle}"></span>
      <span class="book-label" data-text="${label}">${truncateTitle(label)}</span>
    `;
  }

  bookEl.addEventListener('click', () => openBookDetails(book.id));
  bookEl.setAttribute('title', `${book.title}\n${book.author}`);

  if (book.visualWidth < 48) {
    bookEl.setAttribute('data-compact', 'true');
  }

  return bookEl;
}

function openBookDetails(bookId) {
  const book = state.books.find((item) => item.id === bookId);
  if (!book) {
    showToast('Book not found.');
    return;
  }

  elements.bookDetailContent.innerHTML = `
    <div class="detail-cover">
      <img src="${book.coverImage || createGeneratedCover(book)}" alt="Cover of ${escapeHtml(book.title)}" />
    </div>

    <div class="detail-info">
      <div class="detail-heading">
        <h3>${escapeHtml(book.title)}</h3>
        <p>${escapeHtml(book.author)}</p>
      </div>

      <div class="detail-meta">
        <span class="meta-pill">${escapeHtml(book.genre || 'Other')}</span>
        <span class="meta-pill">${escapeHtml(formatRating(book.rating))} / 10</span>
      </div>

      <p class="detail-description">${escapeHtml(book.description || 'No description provided.')}</p>

      <div class="metric-stack">
        <div class="metric-row">
          <label>Reading Progress</label>
          <div class="progress-bar" aria-label="Reading progress ${book.progress || 0}%">
            <span style="width: ${Math.min(100, Math.max(0, Number(book.progress || 0)))}%"></span>
          </div>
          <div class="rating-line">${book.progress || 0}%</div>
        </div>

        <div class="metric-row">
          <label>Rating</label>
          <div class="rating-line">${formatRating(book.rating)} / 10</div>
        </div>
      </div>

      <div class="dimensions-grid">
        <div class="dimension-item"><strong>Pages</strong>${Number(book.pages || 0)}</div>
        <div class="dimension-item"><strong>Height</strong>${Number(book.height || 0)} mm</div>
        <div class="dimension-item"><strong>Width</strong>${Number(book.width || 0)} mm</div>
        <div class="dimension-item"><strong>Thickness</strong>${Number(getEffectiveThickness(book))} mm</div>
      </div>

      <div class="detail-actions">
        <button type="button" class="secondary-button" id="detailEditBtn">EDIT BOOK</button>
        <button type="button" class="danger-button" id="detailDeleteBtn">DELETE BOOK</button>
        <button type="button" class="primary-button" data-close-modal="true">CLOSE</button>
      </div>
    </div>
  `;

  const editButton = document.getElementById('detailEditBtn');
  const deleteButton = document.getElementById('detailDeleteBtn');

  editButton.addEventListener('click', () => {
    closeModals();
    openEditBookModal(book.id);
  });

  deleteButton.addEventListener('click', () => {
    closeModals();
    openDeleteConfirm(book.id);
  });

  elements.bookDetailModal.classList.remove('hidden');
  elements.bookDetailModal.setAttribute('aria-hidden', 'false');
}

function closeModals() {
  elements.bookModal.classList.add('hidden');
  elements.bookDetailModal.classList.add('hidden');
  elements.confirmModal.classList.add('hidden');
  elements.bookModal.setAttribute('aria-hidden', 'true');
  elements.bookDetailModal.setAttribute('aria-hidden', 'true');
  elements.confirmModal.setAttribute('aria-hidden', 'true');
}

function openAddBookModal() {
  state.editingBookId = null;
  elements.bookModalTitle.textContent = 'Add Book';
  elements.bookForm.reset();
  elements.bookColor.value = '#8b3a2d';
  elements.colorLabel.textContent = '#8b3a2d';
  elements.coverPreview.src = ''; 
  elements.spinePreview.src = '';
  elements.bookId.value = '';
  elements.formErrors.textContent = '';
  elements.bookModal.classList.remove('hidden');
  elements.bookModal.setAttribute('aria-hidden', 'false');
  elements.bookShelfPosition.value = 'auto';
}

function openEditBookModal(bookId) {
  const book = state.books.find((item) => item.id === bookId);
  if (!book) {
    showToast('Unable to find the book to edit.');
    return;
  }

  state.editingBookId = book.id;
  elements.bookModalTitle.textContent = 'Edit Book';
  elements.bookId.value = book.id;
  elements.bookTitle.value = book.title || '';
  elements.bookAuthor.value = book.author || '';
  elements.bookGenre.value = book.genre || '';
  elements.bookDescription.value = book.description || '';
  elements.bookPages.value = book.pages || '';
  elements.bookHeight.value = book.height || '';
  elements.bookWidth.value = book.width || '';
  elements.bookThickness.value = book.thickness || '';
  elements.bookProgress.value = book.progress || 0;
  elements.bookRating.value = book.rating || 0;
  elements.bookColor.value = book.color || '#8b3a2d';
  elements.colorLabel.textContent = book.color || '#8b3a2d';
  elements.bookShelfPosition.value = book.shelfPosition || 'auto';
  elements.dimensionUnit.value = book.dimensionUnit || 'mm';

  if (book.coverImage) {
    elements.coverPreview.src = book.coverImage;
  } else {
    elements.coverPreview.src = createGeneratedCover(book);
  }

  if (book.spineImage) {
    elements.spinePreview.src = book.spineImage;
  } else {
    elements.spinePreview.src = createGeneratedSpine(book);
  }

  elements.formErrors.textContent = '';
  elements.bookModal.classList.remove('hidden');
  elements.bookModal.setAttribute('aria-hidden', 'false');
}

function handleBookFormSubmit(event) {
  event.preventDefault();

  const payload = getBookFormData();
  const validationErrors = validateBook(payload);

  if (validationErrors.length) {
    elements.formErrors.textContent = validationErrors.join(' ');
    return;
  }

  if (state.editingBookId) {
    const book = state.books.find((item) => item.id === state.editingBookId);
    if (book) {
      Object.assign(book, payload);
      book.updatedAt = new Date().toISOString();
      showToast('Book updated.');
    }
  } else {
    const newBook = {
      ...payload,
      id: createId(),
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
    state.books.push(newBook);
    showToast('Book added to your library.');
  }

  saveToLocalStorage();
  closeModals();
  renderApp();
}

function getBookFormData() {
  const formData = new FormData(elements.bookForm);
  const thicknessValue = Number(formData.get('thickness') || 0);
  const pages = Number(formData.get('pages') || 0);
  const height = Number(formData.get('height') || 0);
  const width = Number(formData.get('width') || 0);
  const progress = Number(formData.get('progress') || 0);
  const rating = Number(formData.get('rating') || 0);
  const unit = String(formData.get('dimensionUnit') || 'mm');
  const rawThickness = Number.isFinite(thicknessValue) && thicknessValue > 0 ? thicknessValue : calculateEstimatedThickness(pages);

  const effectiveHeight = normalizeDimension(height, unit);
  const effectiveWidth = normalizeDimension(width, unit);
  const effectiveThickness = normalizeDimension(rawThickness, unit);

  const payload = {
    id: elements.bookId.value || state.editingBookId || createId(),
    title: (formData.get('title') || '').trim(),
    author: (formData.get('author') || '').trim(),
    genre: (formData.get('genre') || '').trim() || 'Other',
    description: (formData.get('description') || '').trim(),
    coverImage: elements.coverPreview.dataset.value || '',
    spineImage: elements.spinePreview.dataset.value || '',
    color: String(elements.bookColor.value || '#8b3a2d'),
    pages,
    height: effectiveHeight,
    width: effectiveWidth,
    thickness: effectiveThickness,
    estimatedThickness: calculateEstimatedThickness(pages),
    progress: clamp(progress, 0, 100),
    rating: clamp(rating, 0, 10),
    shelfPosition: String(formData.get('shelfPosition') || 'auto'),
    dimensionUnit: unit,
    createdAt: state.editingBookId ? state.books.find((book) => book.id === state.editingBookId)?.createdAt || new Date().toISOString() : new Date().toISOString(),
  };

  return payload;
}

function validateBook(book) {
  const errors = [];

  if (!book.title) errors.push('Title is required.');
  if (!book.author) errors.push('Author is required.');
  if (!book.genre) errors.push('Genre is required.');
  if (!Number.isFinite(book.pages) || book.pages <= 0) errors.push('Pages must be a positive number.');
  if (!Number.isFinite(book.height) || book.height <= 0) errors.push('Height must be a positive number.');
  if (!Number.isFinite(book.width) || book.width <= 0) errors.push('Width must be a positive number.');
  if (book.progress < 0 || book.progress > 100) errors.push('Reading progress must be between 0 and 100.');
  if (book.rating < 0 || book.rating > 10) errors.push('Rating must be between 0 and 10.');

  return errors;
}

async function handleImageUpload(type, event) {
  const [file] = event.target.files;
  if (!file) return;

  try {
    const compressed = await compressImage(file);
    const preview = type === 'cover' ? elements.coverPreview : elements.spinePreview;
    preview.src = compressed;
    preview.dataset.value = compressed;
    if (type === 'cover') {
      elements.coverPreview.alt = 'Cover preview';
    } else {
      elements.spinePreview.alt = 'Spine preview';
    }
    showToast(`${type === 'cover' ? 'Cover' : 'Spine'} image ready.`);
  } catch (error) {
    showToast(error.message || 'Unable to process image.');
  }
}

function compressImage(file) {
  return new Promise((resolve, reject) => {
    if (!file || !file.type.startsWith('image/')) {
      reject(new Error('Invalid file. Please upload an image.'));
      return;
    }

    const allowed = ['image/jpeg', 'image/png', 'image/webp'];
    if (!allowed.includes(file.type)) {
      reject(new Error('Unsupported image type. Use JPG, PNG, or WebP.'));
      return;
    }

    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const scale = Math.min(1, 1000 / Math.max(img.width, img.height));
        canvas.width = Math.max(1, Math.round(img.width * scale));
        canvas.height = Math.max(1, Math.round(img.height * scale));

        const context = canvas.getContext('2d');
        context.fillStyle = '#f2e9db';
        context.fillRect(0, 0, canvas.width, canvas.height);
        context.drawImage(img, 0, 0, canvas.width, canvas.height);

        const mimeType = file.type === 'image/png' ? 'image/webp' : 'image/jpeg';
        const quality = 0.8;

        canvas.toBlob((blob) => {
          if (!blob) {
            reject(new Error('Image processing failed.'));
            return;
          }

          const blobReader = new FileReader();
          blobReader.onload = () => resolve(blobReader.result);
          blobReader.onerror = () => reject(new Error('Image could not be read.'));
          blobReader.readAsDataURL(blob);
        }, mimeType, quality);
      };

      img.onerror = () => reject(new Error('The selected image is invalid.'));
      img.src = String(reader.result);
    };

    reader.onerror = () => reject(new Error('Unable to read the selected image.'));
    reader.readAsDataURL(file);
  });
}

function saveToLocalStorage() {
  if (!Array.isArray(state.books)) {
    state.books = [];
  }

  localStorage.setItem(STORAGE_KEY, JSON.stringify(state.books));
}

function loadFromLocalStorage() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) {
      state.books = createSampleBooks();
      saveToLocalStorage();
      return;
    }

    const parsed = JSON.parse(raw);
    if (!Array.isArray(parsed)) {
      throw new Error('Library data is malformed.');
    }

    const cleaned = parsed
      .map(normalizeBookData)
      .filter(Boolean);

    state.books = cleaned.length ? cleaned : createSampleBooks();
    saveToLocalStorage();
  } catch (error) {
    console.warn('Corrupted library data detected:', error);
    state.books = createSampleBooks();
    saveToLocalStorage();
    showToast('Stored library data was corrupted, so a fresh shelf was loaded.');
  }
}

function normalizeBookData(book) {
  if (!book || typeof book !== 'object') return null;

  const title = String(book.title || '').trim();
  const author = String(book.author || '').trim();
  const genre = String(book.genre || 'Other').trim() || 'Other';
  const pages = Number(book.pages || 0);
  const height = Number(book.height || 0);
  const width = Number(book.width || 0);

  if (!title || !author || !genre || !pages || !height || !width) {
    return null;
  }

  return {
    id: String(book.id || createId()),
    title,
    author,
    genre,
    description: String(book.description || ''),
    coverImage: String(book.coverImage || ''),
    spineImage: String(book.spineImage || ''),
    color: String(book.color || '#8b3a2d'),
    pages: Math.max(1, Number.isFinite(pages) ? pages : 1),
    height: Math.max(1, Number.isFinite(height) ? height : 1),
    width: Math.max(1, Number.isFinite(width) ? width : 1),
    thickness: Number(book.thickness || calculateEstimatedThickness(Math.max(1, pages))),
    estimatedThickness: Number(book.estimatedThickness || calculateEstimatedThickness(Math.max(1, pages))),
    progress: clamp(Number(book.progress || 0), 0, 100),
    rating: clamp(Number(book.rating || 0), 0, 10),
    shelfPosition: String(book.shelfPosition || 'auto'),
    dimensionUnit: String(book.dimensionUnit || 'mm'),
    createdAt: String(book.createdAt || new Date().toISOString()),
    updatedAt: String(book.updatedAt || book.createdAt || new Date().toISOString()),
  };
}

function createSampleBooks() {
  const sampleBooks = [
    createBookObject({
      title: 'The Hobbit',
      author: 'J.R.R. Tolkien',
      genre: 'Fantasy',
      description: 'A gentle adventure into Middle-earth, where courage grows quietly and the journey matters more than the destination.',
      pages: 310,
      height: 210,
      width: 140,
      thickness: 24,
      progress: 80,
      rating: 9.5,
      color: '#a74536',
      coverImage: createGeneratedCover({ title: 'The Hobbit', color: '#a74536' }),
      spineImage: '',
    }),
    createBookObject({
      title: 'The Little Prince',
      author: 'Antoine de Saint-Exupéry',
      genre: 'Fiction',
      description: 'A poetic adventure about wonder, memory, and the quiet truths that survive adulthood.',
      pages: 96,
      height: 180,
      width: 112,
      thickness: 16,
      progress: 50,
      rating: 9,
      color: '#d18a44',
      coverImage: createGeneratedCover({ title: 'The Little Prince', color: '#d18a44' }),
      spineImage: '',
    }),
    createBookObject({
      title: '1984',
      author: 'George Orwell',
      genre: 'Dystopian',
      description: 'An unsettling portrait of surveillance, power, and the price of truth in a rigid authoritarian state.',
      pages: 328,
      height: 220,
      width: 150,
      thickness: 25,
      progress: 65,
      rating: 9.5,
      color: '#7c4b3a',
      coverImage: createGeneratedCover({ title: '1984', color: '#7c4b3a' }),
      spineImage: '',
    }),
    createBookObject({
      title: 'Pride and Prejudice',
      author: 'Jane Austen',
      genre: 'Romance',
      description: 'A sharp, elegant story of manners, wit, and a very inspiring misunderstanding.',
      pages: 432,
      height: 210,
      width: 140,
      thickness: 28,
      progress: 70,
      rating: 8.5,
      color: '#b56a4d',
      coverImage: createGeneratedCover({ title: 'Pride and Prejudice', color: '#b56a4d' }),
      spineImage: '',
    }),
  ];

  return sampleBooks;
}

function createBookObject(book) {
  const pages = Number(book.pages || 0);
  const height = Number(book.height || 0);
  const width = Number(book.width || 0);
  const thickness = Number(book.thickness || calculateEstimatedThickness(pages));

  return {
    id: createId(),
    title: book.title || 'Untitled',
    author: book.author || 'Unknown Author',
    genre: book.genre || 'Other',
    description: book.description || '',
    coverImage: book.coverImage || '',
    spineImage: book.spineImage || '',
    color: book.color || '#8b3a2d',
    pages,
    height,
    width,
    thickness,
    estimatedThickness: calculateEstimatedThickness(pages),
    progress: clamp(Number(book.progress || 0), 0, 100),
    rating: clamp(Number(book.rating || 0), 0, 10),
    shelfPosition: book.shelfPosition || 'auto',
    dimensionUnit: 'mm',
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  };
}

function calculateEstimatedThickness(pages) {
  const estimated = pages * PAGE_THICKNESS_MM;
  return clamp(estimated, MIN_THICKNESS_MM, MAX_THICKNESS_MM);
}

function calculateBookVisualWidth(book) {
  const thicknessValue = Number(book.thickness || getEffectiveThickness(book));
  const normalized = thicknessValue * 3.1 + 18;
  return clamp(normalized, 22, 120);
}

function calculateBookVisualHeight(book) {
  const rawHeight = Number(book.height || 200);
  const normalized = (rawHeight / 250) * 180;
  return clamp(normalized, 110, 220);
}

function getEffectiveThickness(book) {
  if (Number(book.thickness) > 0) return Number(book.thickness);
  const estimated = Number(book.estimatedThickness || calculateEstimatedThickness(Number(book.pages || 0)));
  return estimated;
}

function normalizeDimension(value, unit) {
  if (!Number.isFinite(value) || value <= 0) return 0;
  if (unit === 'cm') return Math.round(value * 10);
  return Math.round(value);
}

function createGeneratedCover(book) {
  const title = (book.title || 'Book').slice(0, 22);
  const color = book.color || '#a74f37';
  const accent = adjustColor(color, 20);
  const svg = `
    <svg xmlns="http://www.w3.org/2000/svg" width="700" height="1000" viewBox="0 0 700 1000">
      <defs>
        <linearGradient id="g" x1="0" x2="1">
          <stop offset="0%" stop-color="${color}"/>
          <stop offset="100%" stop-color="${accent}"/>
        </linearGradient>
      </defs>
      <rect width="700" height="1000" fill="#efe4d0"/>
      <rect x="40" y="40" width="620" height="920" rx="18" fill="url(#g)"/>
      <rect x="70" y="70" width="560" height="860" rx="14" fill="none" stroke="rgba(255,255,255,0.45)"/>
      <path d="M120 300 C 220 200, 480 200, 580 300 L 580 760 C 480 870, 220 870, 120 760 Z" fill="rgba(255,255,255,0.18)"/>
      <line x1="110" y1="160" x2="590" y2="160" stroke="rgba(255,255,255,0.4)" stroke-width="4"/>
      <line x1="110" y1="820" x2="590" y2="820" stroke="rgba(0,0,0,0.18)" stroke-width="3"/>
      <text x="350" y="470" text-anchor="middle" fill="#fdf6ee" font-size="60" font-family="Georgia, serif" font-weight="700">${escapeForSvg(title)}</text>
      <text x="350" y="620" text-anchor="middle" fill="rgba(255,250,245,0.9)" font-size="24" letter-spacing="5" font-family="Arial, sans-serif">${(book.author || 'Author').slice(0, 24).toUpperCase()}</text>
    </svg>
  `;

  return `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`;
}

function createGeneratedSpine(book) {
  const title = (book.title || 'Book').slice(0, 18);
  const color = book.color || '#8b3a2d';
  const svg = `
    <svg xmlns="http://www.w3.org/2000/svg" width="200" height="800" viewBox="0 0 200 800">
      <defs>
        <linearGradient id="spineGrad" x1="0" x2="1">
          <stop offset="0%" stop-color="${color}"/>
          <stop offset="100%" stop-color="${adjustColor(color, 20)}"/>
        </linearGradient>
      </defs>
      <rect width="200" height="800" fill="url(#spineGrad)"/>
      <rect x="10" y="18" width="180" height="764" fill="none" stroke="rgba(255,255,255,0.35)"/>
      <line x1="22" y1="42" x2="178" y2="42" stroke="rgba(255,255,255,0.24)"/>
      <line x1="22" y1="758" x2="178" y2="758" stroke="rgba(0,0,0,0.18)"/>
      <text x="100" y="420" transform="rotate(90 100 420)" text-anchor="middle" fill="rgba(255,255,255,0.9)" font-family="Georgia, serif" font-size="24" font-weight="700">${escapeForSvg(title)}</text>
    </svg>
  `;

  return `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`;
}

function adjustColor(hex, amount) {
  const h = hex.replace('#', '');
  const num = parseInt(h, 16);
  let r = (num >> 16) + amount;
  let g = ((num >> 8) & 0x00FF) + amount;
  let b = (num & 0x0000FF) + amount;
  r = clamp(r, 0, 255);
  g = clamp(g, 0, 255);
  b = clamp(b, 0, 255);
  return `rgb(${r}, ${g}, ${b})`;
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function createId() {
  if (window.crypto && crypto.randomUUID) {
    return crypto.randomUUID();
  }
  return `book-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

function openDeleteConfirm(bookId) {
  state.selectedBookId = bookId;
  elements.confirmModal.classList.remove('hidden');
  elements.confirmModal.setAttribute('aria-hidden', 'false');
}

function closeDeleteModal() {
  state.selectedBookId = null;
  elements.confirmModal.classList.add('hidden');
  elements.confirmModal.setAttribute('aria-hidden', 'true');
}

function confirmDeleteBook() {
  if (!state.selectedBookId) {
    closeDeleteModal();
    return;
  }

  const index = state.books.findIndex((book) => book.id === state.selectedBookId);
  if (index >= 0) {
    state.books.splice(index, 1);
    saveToLocalStorage();
    showToast('Book removed.');
  }

  state.selectedBookId = null;
  closeDeleteModal();
  renderApp();
}

function exportLibrary() {
  const data = JSON.stringify(state.books, null, 2);
  const blob = new Blob([data], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = 'library-export.json';
  document.body.appendChild(link);
  link.click();
  link.remove();
  URL.revokeObjectURL(url);
  showToast('Library exported.');
}

async function importLibraryFromFile(event) {
  const file = event.target.files?.[0];
  if (!file) return;

  try {
    const text = await file.text();
    const parsed = JSON.parse(text);
    if (!Array.isArray(parsed)) {
      throw new Error('Imported file must contain a JSON array.');
    }

    const repaired = parsed.map(normalizeBookData).filter(Boolean);
    if (!repaired.length) {
      throw new Error('No valid books were found in the imported file.');
    }

    state.books = repaired;
    saveToLocalStorage();
    renderApp();
    showToast('Library imported successfully.');
  } catch (error) {
    console.error(error);
    showToast(error.message || 'Invalid JSON import.');
  } finally {
    event.target.value = '';
  }
}

function clearSampleBooks() {
  state.books = [];
  saveToLocalStorage();
  renderApp();
  showToast('Sample books cleared.');
}

function showToast(message) {
  const toast = document.createElement('div');
  toast.className = 'toast';
  toast.textContent = message;
  elements.toastContainer.appendChild(toast);

  setTimeout(() => {
    toast.remove();
  }, 2200);
}

function createAutumnLeaves() {
  const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  if (reducedMotion) return;

  const colors = ['var(--leaf-amber)', 'var(--leaf-burnt)', 'var(--leaf-gold)', 'var(--leaf-olive)'];
  const leafCount = 14;

  for (let i = 0; i < leafCount; i += 1) {
    const leaf = document.createElement('span');
    leaf.className = 'leaf';
    leaf.style.setProperty('--x', `${Math.random() * 100}%`);
    leaf.style.setProperty('--size', `${12 + Math.random() * 22}px`);
    leaf.style.setProperty('--rotation', `${Math.random() * 220 - 80}deg`);
    leaf.style.setProperty('--duration', `${10 + Math.random() * 9}s`);
    leaf.style.setProperty('--delay', `${Math.random() * 8}s`);
    leaf.style.setProperty('--leaf-color', colors[Math.floor(Math.random() * colors.length)]);
    elements.leafLayer.appendChild(leaf);
  }
}

function generateBookRotation(position, shelfIndex) {
  const offset = shelfIndex === 0 ? 4 : shelfIndex === 1 ? 2 : -2;
  return offset + position % 3 * 2.5;
}

function generateBookOffset(position) {
  return (position % 5) * 1.2;
}

function escapeHtml(value) {
  return String(value)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}

function truncateTitle(title) {
  return title.length > 18 ? `${title.slice(0, 18)}…` : title;
}

function escapeForSvg(value) {
  return String(value)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;');
}

function formatRating(value) {
  const safe = Number(value || 0);
  return safe % 1 === 0 ? `${safe.toFixed(1)}` : safe.toFixed(1);
}

function getDisplayBookColor(book) {
  return book.color || '#8b3a2d';
}

function openMenu() {
  elements.libraryDropdown.classList.remove('hidden');
  elements.libraryMenuButton.setAttribute('aria-expanded', 'true');
}

window.addEventListener('load', () => {
  const menuButton = document.getElementById('libraryMenuButton');
  if (menuButton) {
    menuButton.addEventListener('click', openMenu);
  }
});

function createGeneratedSpinePreview(book) {
  return createGeneratedSpine(book);
}

function createGeneratedCoverPreview(book) {
  return createGeneratedCover(book);
}
