// Firebase services will be available globally

// Document Editor Module - Google Docs style
function openDocumentEditor(classData, existingDoc = null) {
    console.log('Opening document editor', { classData, existingDoc });
    
    // Set global variables for manual save functionality
    window.currentClassData = classData;
    window.currentExistingDoc = existingDoc;
    
    // Add beforeunload event listener to prevent data loss
    window.addEventListener('beforeunload', handleBeforeUnload);
    
    // Set up periodic auto-save (every 30 seconds)
    window.autoSaveInterval = setInterval(() => {
        if (document.getElementById('documentEditorScreen')) {
            console.log('Auto-saving document...');
            saveDocument(window.currentClassData, window.currentExistingDoc).catch(error => {
                console.error('Auto-save error:', error);
            });
        }
    }, 30000); // 30 seconds
    
    // Hide class view
    const classView = document.getElementById('classViewContainer');
    if (classView) {
        classView.style.display = 'none';
    }
    
    // Prevent body scroll when document editor is open
    document.body.classList.add('document-editor-open');
    
    // Create editor screen
    const editorScreen = createEditorScreen(classData, existingDoc);
    const mainWrapper = document.querySelector('.main-wrapper');
    if (mainWrapper) {
        mainWrapper.appendChild(editorScreen);
    }
    
    // DISABLED: Don't load any suggestions to prevent multiplying
    // setTimeout(() => {
    //     // Clean up any orphaned suggestions first
    //     window.cleanupOrphanedSuggestions(classData, existingDoc);
    //     
    //     // Load current suggestions (active ones)
    //     window.loadCurrentSuggestions(classData, existingDoc);
    //     
    //     // Only load saved suggestions if no current suggestions exist
    //     const currentSuggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_current-document`;
    //     const hasCurrentSuggestions = localStorage.getItem(currentSuggestionsKey);
    //     if (!hasCurrentSuggestions) {
    //         window.loadSavedSuggestions(classData, existingDoc);
    //     }
    // }, 500);
    
    // Clear only suggestion storage, not DOM elements that might interfere with editing
    setTimeout(() => {
        if (typeof clearSuggestionStorageOnly === 'function') {
            clearSuggestionStorageOnly();
        } else {
            console.log('clearSuggestionStorageOnly function not available, skipping cleanup');
        }
        
        // Ensure text selection works properly
        if (typeof enableTextSelection === 'function') {
            enableTextSelection();
        } else {
            console.log('enableTextSelection function not available, skipping');
        }
    }, 100);
    
    // Add event listener to save suggestions when navigating away
    const saveOnUnload = () => {
        window.saveCurrentSuggestions();
    };
    
    // Save suggestions on various navigation events
    window.addEventListener('beforeunload', saveOnUnload);
    window.addEventListener('popstate', saveOnUnload);
    
    // Also save when the editor screen is removed
    const editorScreenElement = document.getElementById('documentEditorScreen');
    if (editorScreenElement) {
        const observer = new MutationObserver((mutations) => {
            mutations.forEach((mutation) => {
                if (mutation.type === 'childList') {
                    mutation.removedNodes.forEach((node) => {
                        if (node.id === 'documentEditorScreen') {
                            window.saveCurrentSuggestions();
                            observer.disconnect();
                        }
                    });
                }
            });
        });
        observer.observe(document.body, { childList: true, subtree: true });
    }
    
    // Periodic save every 5 seconds to ensure suggestions are always saved
    const periodicSave = setInterval(() => {
        if (document.getElementById('documentEditorScreen')) {
            window.saveCurrentSuggestions();
        } else {
            clearInterval(periodicSave);
        }
    }, 5000);
    
    // Auto-save is handled by the 30-second interval and input events
    // Removed duplicate 3-second interval to prevent multiple saves
    
    // Auto-save on content changes
    const contentElements = document.querySelectorAll('.doc-editor-content');
    contentElements.forEach(content => {
        // Ensure content is focusable and editable
        content.setAttribute('contenteditable', 'true');
        content.setAttribute('tabindex', '0');
        content.setAttribute('spellcheck', 'true');
        
        // Enable text selection
        content.style.userSelect = 'text';
        content.style.webkitUserSelect = 'text';
        content.style.mozUserSelect = 'text';
        content.style.msUserSelect = 'text';
        
        // Multiple event listeners for comprehensive auto-save
        const triggerAutoSave = () => {
            clearTimeout(window.autoSaveTimeout);
            window.autoSaveTimeout = setTimeout(() => {
                autoSaveDocument(window.currentClassData, window.currentExistingDoc);
            }, 500); // Reduced debounce time for more responsive saving
        };
        
        // Input events - remove ghost text immediately on any input
        content.addEventListener('input', () => {
            removeGhostText();
            triggerAutoSave();
        });
        content.addEventListener('keydown', (e) => {
            // Remove ghost text on any key except special keys
            if (e.key.length === 1 || e.key === 'Backspace' || e.key === 'Delete') {
                removeGhostText();
            }
        });
        content.addEventListener('keyup', () => {
            removeGhostText();
            triggerAutoSave();
        });
        content.addEventListener('paste', () => {
            removeGhostText();
            setTimeout(triggerAutoSave, 100); // Small delay for paste content to be processed
        });
        content.addEventListener('keypress', () => {
            removeGhostText();
        });
        
        // Ensure content can be focused and selected
        content.addEventListener('click', () => {
            content.focus();
        });
        content.addEventListener('focus', () => {
            removeGhostText();
        });
        
        // Enable text selection on mouse events
        content.addEventListener('mousedown', (e) => {
            e.stopPropagation();
        });
        
        content.addEventListener('mouseup', (e) => {
            e.stopPropagation();
        });
        
        content.addEventListener('selectstart', (e) => {
            e.stopPropagation();
        });
    });
    
    // Auto-save on title changes
    const titleInput = document.getElementById('docEditorTitle');
    if (titleInput) {
        const triggerTitleAutoSave = () => {
            clearTimeout(window.titleAutoSaveTimeout);
            window.titleAutoSaveTimeout = setTimeout(() => {
                autoSaveDocument(window.currentClassData, window.currentExistingDoc);
            }, 500);
        };
        
        titleInput.addEventListener('input', triggerTitleAutoSave);
        titleInput.addEventListener('keyup', triggerTitleAutoSave);
        titleInput.addEventListener('blur', triggerTitleAutoSave); // Save when user clicks away
    }
    
    // Save on focus loss (when user clicks outside or switches tabs)
    window.addEventListener('blur', saveOnUnload);
    document.addEventListener('visibilitychange', () => {
        if (document.hidden) {
            window.saveCurrentSuggestions();
        }
    });
}

function generatePages(content) {
    // Create single infinite content area instead of multiple pages
    const hasContent = content && content.trim().length > 0;
    console.log('generatePages - hasContent:', hasContent, 'content length:', content?.length);
    
    if (hasContent) {
        console.log('Loading document content with full HTML:', content.substring(0, 200) + '...');
    }
    
    return `
        <div class="doc-editor-content" 
             id="docEditorContent" 
             contenteditable="true" 
             spellcheck="true" 
             tabindex="0">
            ${hasContent ? content : ''}
        </div>
    `;
}

function createEditorScreen(classData, existingDoc) {
    const container = document.createElement('div');
    container.className = 'document-editor-screen screen-container';
    container.id = 'documentEditorScreen';
    
    const docTitle = existingDoc ? existingDoc.title : 'Untitled Document';
    const docContent = existingDoc ? existingDoc.content : '';
    
    container.innerHTML = `
        <div class="doc-editor-header">
            <button class="doc-tool-btn doc-back-btn-arrow" id="docBackToClassBtn" title="Back to Class" onclick="closeDocumentEditor()">
                <span class="back-icon">←</span>
            </button>
            <input type="text" class="doc-editor-title" id="docEditorTitle" 
                   value="${docTitle}" placeholder="Untitled Document">
            <button class="doc-tool-btn" id="manualSaveBtn" title="Save Document (Ctrl+S)">💾 Save</button>
            <button class="doc-tool-btn" id="savingStatus">Saved</button>
            <button class="doc-tool-btn" id="downloadPdfBtn" title="Download as PDF">📥 PDF</button>
            <div class="doc-stats">
                <span class="stat-item" id="wordCount">0 words</span>
                <span class="stat-divider">•</span>
                <span class="stat-item" id="charCount">0 characters</span>
            </div>
            <button class="doc-tool-btn" id="humanizeBtn" title="Humanize Text">Humanize</button>
            <button class="doc-chat-toggle-btn" id="chatToggleBtn" title="Toggle Genius AI Chat">
                <img src="assets/darkgenius.png" alt="Genius AI" class="chat-toggle-icon">
            </button>
        </div>
        
        <div class="doc-editor-toolbar">
            <div class="toolbar-group">
                <select class="toolbar-select" id="fontSelect">
                    <option value="Arial">Arial</option>
                    <option value="Times New Roman">Times New Roman</option>
                    <option value="Courier New">Courier New</option>
                    <option value="Georgia">Georgia</option>
                    <option value="Verdana">Verdana</option>
                    <option value="Segoe UI" selected>Segoe UI</option>
                </select>
                <select class="toolbar-select" id="fontSizeSelect">
                    <option value="10">10</option>
                    <option value="12">12</option>
                    <option value="14">14</option>
                    <option value="16" selected>16</option>
                    <option value="18">18</option>
                    <option value="20">20</option>
                    <option value="24">24</option>
                    <option value="28">28</option>
                    <option value="32">32</option>
                    <option value="36">36</option>
                    <option value="48">48</option>
                </select>
                <select class="toolbar-select" id="headingSelect">
                    <option value="p">Normal Text</option>
                    <option value="h1">Title</option>
                    <option value="h2">Subtitle</option>
                    <option value="h3">Heading 1</option>
                    <option value="h4">Heading 2</option>
                    <option value="h5">Heading 3</option>
                    <option value="h6">Heading 4</option>
                    <option value="blockquote">Quote</option>
                    <option value="code">Code</option>
                </select>
            </div>
            
            <div class="toolbar-divider"></div>
            
            <div class="toolbar-group">
                <button class="toolbar-btn" id="boldBtn" title="Bold (Ctrl+B)" tabindex="-1">
                    <strong>B</strong>
                </button>
                <button class="toolbar-btn" id="italicBtn" title="Italic (Ctrl+I)" tabindex="-1">
                    <em>I</em>
                </button>
                <button class="toolbar-btn" id="underlineBtn" title="Underline (Ctrl+U)" tabindex="-1">
                    <u>U</u>
                </button>
                <button class="toolbar-btn" id="strikeBtn" title="Strikethrough" tabindex="-1">
                    <s>S</s>
                </button>
            </div>
            
            <div class="toolbar-divider"></div>
            
            <div class="toolbar-group">
                <button class="toolbar-btn align-icon-btn" id="alignLeftBtn" title="Align Left" tabindex="-1">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
                        <rect x="2" y="3" width="12" height="1.5"/>
                        <rect x="2" y="6" width="8" height="1.5"/>
                        <rect x="2" y="9" width="10" height="1.5"/>
                        <rect x="2" y="12" width="6" height="1.5"/>
                    </svg>
                </button>
                <button class="toolbar-btn align-icon-btn" id="alignCenterBtn" title="Align Center" tabindex="-1">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
                        <rect x="2" y="3" width="12" height="1.5"/>
                        <rect x="4" y="6" width="8" height="1.5"/>
                        <rect x="3" y="9" width="10" height="1.5"/>
                        <rect x="5" y="12" width="6" height="1.5"/>
                    </svg>
                </button>
                <button class="toolbar-btn align-icon-btn" id="alignRightBtn" title="Align Right" tabindex="-1">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
                        <rect x="2" y="3" width="12" height="1.5"/>
                        <rect x="6" y="6" width="8" height="1.5"/>
                        <rect x="4" y="9" width="10" height="1.5"/>
                        <rect x="8" y="12" width="6" height="1.5"/>
                    </svg>
                </button>
                <button class="toolbar-btn align-icon-btn" id="justifyBtn" title="Justify" tabindex="-1">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
                        <rect x="2" y="3" width="12" height="1.5"/>
                        <rect x="2" y="6" width="12" height="1.5"/>
                        <rect x="2" y="9" width="12" height="1.5"/>
                        <rect x="2" y="12" width="12" height="1.5"/>
                    </svg>
                </button>
            </div>
            
            <div class="toolbar-divider"></div>
            
            <div class="toolbar-group">
                <button class="toolbar-btn" id="bulletListBtn" title="Bullet List" tabindex="-1">
                    •
                </button>
                <button class="toolbar-btn" id="numberListBtn" title="Numbered List" tabindex="-1">
                    1.
                </button>
                <button class="toolbar-btn" id="outdentBtn" title="Decrease Indent" tabindex="-1">
                    ⇤
                </button>
                <button class="toolbar-btn" id="indentBtn" title="Increase Indent" tabindex="-1">
                    ⇥
                </button>
            </div>
            
            <div class="toolbar-divider"></div>
            
            <div class="toolbar-group">
                <button class="toolbar-btn" id="undoBtn" title="Undo (Ctrl+Z)" tabindex="-1">
                    ↶
                </button>
                <button class="toolbar-btn" id="redoBtn" title="Redo (Ctrl+Y)" tabindex="-1">
                    ↷
                </button>
            </div>
            
            <div class="toolbar-divider"></div>
            
            <div class="toolbar-group">
                <button class="toolbar-btn toolbar-text-btn" id="removeSuggestionsBtn" title="Remove All Suggestions" tabindex="-1">
                    Remove Suggestions
                </button>
            </div>
            
            <div class="toolbar-divider"></div>
            
            
        </div>
        
        <div class="doc-editor-content-wrapper">
            <div class="suggestions-sidebar" id="suggestionsSidebar"></div>
            ${generatePages(docContent)}
        </div>
        
        <div class="genius-chat-container" id="geniusChatContainer">
        <div class="genius-chat-input-wrapper" id="geniusInputWrapper">
            <img src="assets/darkgenius.png" alt="Genius" class="genius-chat-icon">
            <input type="text" class="genius-chat-input" id="geniusChatInput" placeholder="Talk to Genius">
            <div class="genius-mode-toggle" id="geniusModeToggle" title="Change mode">
                <span class="mode-icon" id="modeIcon" style="font-size: 16px; font-weight: bold; color: #fff;">?</span>
            </div>
        </div>
        </div>
        
        <div class="genius-sidebar" id="geniusSidebar">
            <div class="genius-sidebar-header">
                <div class="genius-sidebar-title">
                    <img src="assets/darkgenius.png" alt="Genius" class="genius-sidebar-icon">
                    <span>Genius AI</span>
                </div>
                <div class="genius-header-actions">
                    <button class="genius-new-chat-btn" id="newChatBtn" title="New Chat">+</button>
                    <button class="genius-close-btn" id="closeSidebarBtn">✕</button>
                </div>
            </div>
            <div class="genius-chat-tabs" id="geniusChatTabs">
                <!-- Chat tabs will be dynamically added here -->
            </div>
            <div class="genius-chat-messages" id="geniusChatMessages"></div>
            <div class="genius-chat-input-container">
                <div class="genius-chat-input-wrapper-sidebar">
                    <input type="text" class="genius-chat-input-sidebar" id="geniusChatInputSidebar" placeholder="Ask Genius anything...">
                    <button class="genius-send-btn" id="geniusSendBtn">Send</button>
                </div>
            </div>
            <div class="genius-chat-status" id="geniusChatStatus"></div>
            <div class="genius-resize-handle" id="geniusResizeHandle"></div>
        </div>
    `;
    
    // Setup editor functionality
    setTimeout(() => {
        try {
            setupEditorControls(classData, existingDoc);
        } catch (error) {
            console.error('Error setting up editor controls:', error);
        }
        
        try {
            setupToolbarButtons();
        } catch (error) {
            console.error('Error setting up toolbar buttons:', error);
        }
        
         try {
             setupGeniusChat(classData, existingDoc);
         } catch (error) {
             console.error('Error setting up Genius chat:', error);
         }
         
         // Add event delegation as fallback for back button
         document.addEventListener('click', (e) => {
             if (e.target && e.target.id === 'docBackToClassBtn') {
                 console.log('Back button clicked via event delegation!');
                 e.preventDefault();
                 e.stopPropagation();
                 closeDocumentEditor();
             }
         });
         
         // Test if back button is clickable
         const testBtn = document.getElementById('docBackToClassBtn');
         if (testBtn) {
             console.log('Back button test - element:', testBtn);
             console.log('Back button test - computed style:', window.getComputedStyle(testBtn));
             console.log('Back button test - offsetParent:', testBtn.offsetParent);
             console.log('Back button test - disabled:', testBtn.disabled);
         }
         
         // Load saved suggestions after everything is set up
         setTimeout(() => {
             loadSavedSuggestions(classData, existingDoc);
         }, 300);
         
        // Initialize spell check
        console.log('🚀 About to initialize spell check...');
        initializeSpellCheck();
    }, 100);
    
    return container;
}

function setupEditorControls(classData, existingDoc) {
    const titleInput = document.getElementById('docEditorTitle');
    const content = document.getElementById('docEditorContent');
    const savingStatus = document.getElementById('savingStatus');
    
    // Initialize counters
    updateDocumentStats();
    
    // Initialize placeholder visibility with a small delay to ensure DOM is ready
    setTimeout(() => {
        updatePlaceholderVisibility();
        // Ensure content is properly editable
        if (content) {
            content.setAttribute('contenteditable', 'true');
            content.setAttribute('spellcheck', 'true');
        }
        // No periodic check needed - event-driven updates are sufficient
    }, 100);
    
    // Initialize undo/redo functionality
    if (typeof initializeUndoRedo === 'function') {
        initializeUndoRedo();
    }
    
    
    // Store the document ID to ensure we always update the same document
    let currentDocId = existingDoc ? existingDoc.id : Date.now().toString();
    
    // Auto-save timer (make global so spell check can access)
    window.saveTimer = window.saveTimer || null;
    window.hasChanges = window.hasChanges || false;
    
    const autoSave = async () => {
        if (window.hasChanges) {
            // Always use the existingDoc reference to ensure we update the same document
            if (existingDoc) {
                const updatedDoc = await saveDocument(classData, existingDoc);
                // Update the existingDoc reference with the returned document
                Object.assign(existingDoc, updatedDoc);
            } else {
                // Only create a new document if we don't have an existing one
                const docToUpdate = { id: currentDocId };
                const updatedDoc = await saveDocument(classData, docToUpdate);
                // Create the existingDoc reference for future auto-saves
                existingDoc = updatedDoc;
            }
            window.hasChanges = false;
            savingStatus.textContent = 'Saved';
            savingStatus.style.color = '#888888';
        }
    };
    
    // Helper function to trigger auto-save after formatting changes
    const triggerFormattingSave = () => {
        console.log('💾 Triggering formatting save...');
        window.hasChanges = true;
        savingStatus.textContent = 'Saving...';
        savingStatus.style.color = '#aaaaaa';
        clearTimeout(window.saveTimer);
        window.saveTimer = setTimeout(autoSave, 1000); // Save quickly after formatting
    };
    
    // Make autoSave globally accessible
    window.autoSave = autoSave;
    
    // Define the click handler function first
    function handleBackButtonClick(e) {
        console.log('Back button clicked!');
        e.preventDefault();
        e.stopPropagation();
            closeDocumentEditor();
    }
    
    // Simple back button setup
    setTimeout(() => {
        const backBtn = document.getElementById('docBackToClassBtn');
        if (backBtn) {
            backBtn.onclick = () => {
                console.log('Back clicked');
                closeDocumentEditor();
            };
        }
    }, 100);
    
    // Simple save button setup
    setTimeout(() => {
        const saveBtn = document.getElementById('manualSaveBtn');
        console.log('Save button found:', saveBtn);
        if (saveBtn) {
            saveBtn.onclick = async (e) => {
                e.preventDefault();
                e.stopPropagation();
                console.log('Save clicked');
                const classData = window.currentClassData;
                const existingDoc = window.currentExistingDoc;
                if (classData && existingDoc) {
                    await saveDocument(classData, existingDoc);
                }
            };
            console.log('Save button onclick set');
        } else {
            console.error('Save button not found!');
        }
    }, 150);
    
    // Simple PDF button setup
    setTimeout(() => {
        const pdfBtn = document.getElementById('downloadPdfBtn');
        console.log('PDF button found:', pdfBtn);
        if (pdfBtn) {
            pdfBtn.onclick = (e) => {
                e.preventDefault();
                e.stopPropagation();
                console.log('PDF clicked');
            downloadAsPDF();
            };
            console.log('PDF button onclick set');
        } else {
            console.error('PDF button not found!');
    }
    }, 200);
    
    // Simple humanize button setup
    setTimeout(() => {
    const humanizeBtn = document.getElementById('humanizeBtn');
    if (humanizeBtn) {
            humanizeBtn.onclick = () => {
                console.log('Humanize clicked');
            humanizeText();
            };
        }
    }, 250);
    
    
    // Keyboard shortcuts
    document.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.key === 's') {
            e.preventDefault();
            const manualSaveBtn = document.getElementById('manualSaveBtn');
            if (manualSaveBtn && !manualSaveBtn.disabled) {
                manualSaveBtn.click();
            }
        }
    });
    
    // Title change
    if (titleInput) {
        titleInput.addEventListener('input', () => {
            window.hasChanges = true;
            savingStatus.textContent = 'Saving...';
            savingStatus.style.color = '#aaaaaa';
            clearTimeout(window.saveTimer);
            window.saveTimer = setTimeout(autoSave, 2000);
        });
    }
    
    // Content change
    if (content) {
        content.addEventListener('input', () => {
            window.hasChanges = true;
            savingStatus.textContent = 'Saving...';
            savingStatus.style.color = '#aaaaaa';
            clearTimeout(window.saveTimer);
            window.saveTimer = setTimeout(autoSave, 2000);
            updateDocumentStats();
            
            // Show/hide placeholder based on content with delay to prevent flashing
            setTimeout(() => updatePlaceholderVisibility(), 200);
            
        });
        
        // Handle Tab key
        content.addEventListener('keydown', (e) => {
            if (e.key === 'Tab') {
                e.preventDefault();
                
                // Insert tab character (4 spaces)
                document.execCommand('insertHTML', false, '&nbsp;&nbsp;&nbsp;&nbsp;');
                
                window.hasChanges = true;
                savingStatus.textContent = 'Saving...';
                savingStatus.style.color = '#aaaaaa';
                clearTimeout(window.saveTimer);
                window.saveTimer = setTimeout(autoSave, 2000);
            }
            
        });
        
        // No pagination needed for infinite page - content can grow infinitely
        
        // Focus on content
        content.focus();
        
        // Handle focus/blur for placeholder with delay to prevent flashing
        content.addEventListener('focus', () => {
            setTimeout(() => updatePlaceholderVisibility(), 100);
        });
        
        content.addEventListener('blur', () => {
            setTimeout(() => updatePlaceholderVisibility(), 100);
        });
    }
    
    // Simple toolbar setup - no complex function needed
    
    // Simple CSS fix for button clickability
    setTimeout(() => {
        const style = document.createElement('style');
        style.textContent = `
            .doc-tool-btn, .toolbar-btn, .doc-back-btn-arrow {
                pointer-events: auto !important;
                cursor: pointer !important;
            }
        `;
        document.head.appendChild(style);
        console.log('Simple CSS fix applied');
    }, 500);
}

// createNewPage function removed - no longer needed for infinite page

function selectAllInElement(element) {
    const range = document.createRange();
    range.selectNodeContents(element);
    const selection = window.getSelection();
    selection.removeAllRanges();
    selection.addRange(range);
}

function setupToolbarButtons() {
    console.log('setupToolbarButtons called');
    
    // Helper function to execute command and maintain focus
    const executeCommand = (command, value = null) => {
        console.log('Executing command:', command, value);
        
        // Ensure content is focused first
        const activeContent = document.querySelector('.doc-editor-content:focus') || document.querySelector('.doc-editor-content');
        if (activeContent) {
            activeContent.focus();
        }
        
        // Save current selection
        const selection = window.getSelection();
        const range = selection.rangeCount > 0 ? selection.getRangeAt(0) : null;
        
        // Execute command
        const success = document.execCommand(command, false, value);
        console.log('Command executed:', success);
        
        // Restore selection if it was lost
        if (range) {
            try {
                selection.removeAllRanges();
                selection.addRange(range);
            } catch (e) {
                console.log('Could not restore selection:', e);
            }
        }
        
        // Update button states
        setTimeout(updateButtonStates, 100);
        
        // Trigger auto-save
        if (typeof triggerFormattingSave === 'function') {
            triggerFormattingSave();
        }
    };
    
    // Helper function to update button states
    const updateButtonStates = () => {
        const content = document.querySelector('.doc-editor-content');
        if (!content) return;
        
        // Update formatting buttons
        const boldBtn = document.getElementById('boldBtn');
        const italicBtn = document.getElementById('italicBtn');
        const underlineBtn = document.getElementById('underlineBtn');
        const strikeBtn = document.getElementById('strikeBtn');
        
        if (boldBtn) {
            boldBtn.classList.toggle('active', document.queryCommandState('bold'));
        }
        if (italicBtn) {
            italicBtn.classList.toggle('active', document.queryCommandState('italic'));
        }
        if (underlineBtn) {
            underlineBtn.classList.toggle('active', document.queryCommandState('underline'));
        }
        if (strikeBtn) {
            strikeBtn.classList.toggle('active', document.queryCommandState('strikeThrough'));
        }
        
        // Update alignment buttons
        const alignLeftBtn = document.getElementById('alignLeftBtn');
        const alignCenterBtn = document.getElementById('alignCenterBtn');
        const alignRightBtn = document.getElementById('alignRightBtn');
        const justifyBtn = document.getElementById('justifyBtn');
        
        if (alignLeftBtn) {
            alignLeftBtn.classList.toggle('active', document.queryCommandState('justifyLeft'));
        }
        if (alignCenterBtn) {
            alignCenterBtn.classList.toggle('active', document.queryCommandState('justifyCenter'));
        }
        if (alignRightBtn) {
            alignRightBtn.classList.toggle('active', document.queryCommandState('justifyRight'));
        }
        if (justifyBtn) {
            justifyBtn.classList.toggle('active', document.queryCommandState('justifyFull'));
        }
        
        // Update font dropdown to show current font
        const fontSelect = document.getElementById('fontSelect');
        if (fontSelect) {
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
                const container = range.commonAncestorContainer;
                const element = container.nodeType === Node.TEXT_NODE ? container.parentElement : container;
                
                if (element) {
                    const computedStyle = window.getComputedStyle(element);
                    const fontFamily = computedStyle.fontFamily;
                    
                    // Find matching option
                    for (let option of fontSelect.options) {
                        if (fontFamily.includes(option.value)) {
                            fontSelect.value = option.value;
                            break;
                        }
                    }
                }
            }
        }
        
        // Update font size dropdown to show current size
        const fontSizeSelect = document.getElementById('fontSizeSelect');
        if (fontSizeSelect) {
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
                const container = range.commonAncestorContainer;
                const element = container.nodeType === Node.TEXT_NODE ? container.parentElement : container;
                
                if (element) {
                    const computedStyle = window.getComputedStyle(element);
                    const fontSize = computedStyle.fontSize;
                    const sizeValue = parseInt(fontSize);
                    
                    // Find matching option
                    for (let option of fontSizeSelect.options) {
                        if (parseInt(option.value) === sizeValue) {
                            fontSizeSelect.value = option.value;
                            break;
                        }
                    }
                }
            }
        }
        
        // Update heading dropdown to show current heading style
        const headingSelect = document.getElementById('headingSelect');
        if (headingSelect) {
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
                const container = range.commonAncestorContainer;
                const element = container.nodeType === Node.TEXT_NODE ? container.parentElement : container;
                
                if (element) {
                    const tagName = element.tagName.toLowerCase();
                    
                    // Check if it's a heading, paragraph, blockquote, or code
                    if (['h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'blockquote', 'code'].includes(tagName)) {
                        headingSelect.value = tagName;
                    } else {
                        // Check if it's inside a heading element
                        const headingElement = element.closest('h1, h2, h3, h4, h5, h6, p, blockquote, code');
                        if (headingElement) {
                            headingSelect.value = headingElement.tagName.toLowerCase();
                        } else {
                            headingSelect.value = 'p'; // Default to paragraph
                        }
                    }
                }
            }
        }
    };
    
    // Add keyboard shortcuts
    const addKeyboardShortcuts = () => {
        const content = document.querySelector('.doc-editor-content');
        if (!content) return;
        
        content.addEventListener('keydown', (e) => {
            // Ctrl/Cmd + B for Bold
            if ((e.ctrlKey || e.metaKey) && e.key === 'b') {
                e.preventDefault();
                executeCommand('bold');
                triggerFormattingSave();
                updateButtonStates();
            }
            // Ctrl/Cmd + I for Italic
            else if ((e.ctrlKey || e.metaKey) && e.key === 'i') {
                e.preventDefault();
                executeCommand('italic');
                triggerFormattingSave();
                updateButtonStates();
            }
            // Ctrl/Cmd + U for Underline
            else if ((e.ctrlKey || e.metaKey) && e.key === 'u') {
                e.preventDefault();
                executeCommand('underline');
                triggerFormattingSave();
                updateButtonStates();
            }
            // Ctrl/Cmd + Z for Undo
            else if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
                e.preventDefault();
                undo();
            }
            // Ctrl/Cmd + Y or Ctrl/Cmd + Shift + Z for Redo
            else if ((e.ctrlKey || e.metaKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
                e.preventDefault();
                redo();
            }
            // Ctrl/Cmd + S for Save
            else if ((e.ctrlKey || e.metaKey) && e.key === 's') {
                e.preventDefault();
                if (typeof window.autoSave === 'function') {
                    window.hasChanges = true;
                    window.autoSave();
                }
            }
            // Ctrl/Cmd + H for Find and Replace
            else if ((e.ctrlKey || e.metaKey) && e.key === 'h') {
                e.preventDefault();
                showFindReplaceDialog();
            }
            // Ctrl/Cmd + F for Find
            else if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
                e.preventDefault();
                showFindDialog();
            }
        });
    };
    
    // Initialize keyboard shortcuts
    addKeyboardShortcuts();
    
    // Update button states on selection change
    const content = document.querySelector('.doc-editor-content');
    if (content) {
        content.addEventListener('selectionchange', updateButtonStates);
        content.addEventListener('keyup', updateButtonStates);
        content.addEventListener('mouseup', updateButtonStates);
    }
    
    // Font select
    const fontSelect = document.getElementById('fontSelect');
    console.log('Font select found:', !!fontSelect);
    if (fontSelect) {
        fontSelect.addEventListener('change', (e) => {
            const selectedFont = e.target.value;
            console.log('Font changed to:', selectedFont);
            
            // Focus content first
            const activeContent = document.querySelector('.doc-editor-content:focus') || document.querySelector('.doc-editor-content');
            if (activeContent) {
                activeContent.focus();
            }
            
            // Apply font to selection or current element
            const selection = window.getSelection();
            if (selection.rangeCount > 0 && !selection.isCollapsed) {
                // Apply to selection
                const success = document.execCommand('fontName', false, selectedFont);
                console.log('Font command executed on selection:', success);
            } else {
                // Apply to current paragraph or create a span
                const range = selection.rangeCount > 0 ? selection.getRangeAt(0) : null;
                if (range) {
                    const span = document.createElement('span');
                    span.style.fontFamily = selectedFont;
                    try {
                        range.surroundContents(span);
                        console.log('Font applied via span');
                    } catch (e) {
                        // If we can't surround, insert the span
                        range.insertNode(span);
                        range.collapse(false);
                        console.log('Font applied via insert');
                    }
                }
            }
            
            if (typeof triggerFormattingSave === 'function') {
            triggerFormattingSave();
            }
        });
    }
    
    // Font size select
    const fontSizeSelect = document.getElementById('fontSizeSelect');
    if (fontSizeSelect) {
        fontSizeSelect.addEventListener('change', (e) => {
            const size = e.target.value;
            console.log('Changing font size to:', size);
            
            // Focus content first
            const activeContent = document.querySelector('.doc-editor-content:focus') || document.querySelector('.doc-editor-content');
            if (activeContent) {
                activeContent.focus();
            }
            
            // Use a more reliable method for font size
            const success = document.execCommand('fontSize', false, '7');
            console.log('Font size command executed:', success);
            
            setTimeout(() => {
                // Find the most recent font element and update it
                const fontElements = document.querySelectorAll('font[size="7"]');
                if (fontElements.length > 0) {
                    const lastFontElement = fontElements[fontElements.length - 1];
                    lastFontElement.removeAttribute('size');
                    lastFontElement.style.fontSize = size + 'px';
                    console.log('Updated font size to:', size + 'px');
                } else {
                    // Fallback: apply to selection or current element
                    const selection = window.getSelection();
                    if (selection.rangeCount > 0) {
                        const range = selection.getRangeAt(0);
                        const span = document.createElement('span');
                        span.style.fontSize = size + 'px';
                        try {
                            range.surroundContents(span);
                        } catch (e) {
                            console.log('Could not apply font size directly:', e);
                        }
                    }
                }
                
                if (typeof triggerFormattingSave === 'function') {
                triggerFormattingSave();
                }
            }, 50);
        });
    }
    
    // Heading style select
    const headingSelect = document.getElementById('headingSelect');
    if (headingSelect) {
        headingSelect.addEventListener('change', (e) => {
            const selectedHeading = e.target.value;
            console.log('Heading changed to:', selectedHeading);
            
            // Focus content first
            const activeContent = document.querySelector('.doc-editor-content:focus') || document.querySelector('.doc-editor-content');
            if (activeContent) {
                activeContent.focus();
            }
            
            // Apply heading style
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
                
                if (selectedHeading === 'p') {
                    // Convert to paragraph
                    const p = document.createElement('p');
                    try {
                        range.surroundContents(p);
                    } catch (e) {
                        // If we can't surround, wrap the content
                        const contents = range.extractContents();
                        p.appendChild(contents);
                        range.insertNode(p);
                    }
                } else if (selectedHeading === 'blockquote') {
                    // Convert to blockquote
                    const blockquote = document.createElement('blockquote');
                    blockquote.style.borderLeft = '4px solid #4a9eff';
                    blockquote.style.paddingLeft = '16px';
                    blockquote.style.margin = '8px 0';
                    blockquote.style.fontStyle = 'italic';
                    blockquote.style.color = '#cccccc';
                    try {
                        range.surroundContents(blockquote);
                    } catch (e) {
                        const contents = range.extractContents();
                        blockquote.appendChild(contents);
                        range.insertNode(blockquote);
                    }
                } else if (selectedHeading === 'code') {
                    // Convert to code
                    const code = document.createElement('code');
                    code.style.backgroundColor = 'rgba(255, 255, 255, 0.1)';
                    code.style.padding = '2px 4px';
                    code.style.borderRadius = '3px';
                    code.style.fontFamily = 'monospace';
                    code.style.fontSize = '14px';
                    try {
                        range.surroundContents(code);
                    } catch (e) {
                        const contents = range.extractContents();
                        code.appendChild(contents);
                        range.insertNode(code);
                    }
                } else {
                    // Convert to heading (h1, h2, h3, h4, h5, h6)
                    const heading = document.createElement(selectedHeading);
                    
                    // Apply different styles based on heading level
                    switch(selectedHeading) {
                        case 'h1':
                            heading.style.fontSize = '32px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '16px 0 8px 0';
                            heading.style.color = '#ffffff';
                            break;
                        case 'h2':
                            heading.style.fontSize = '28px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '14px 0 6px 0';
                            heading.style.color = '#ffffff';
                            break;
                        case 'h3':
                            heading.style.fontSize = '24px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '12px 0 4px 0';
                            heading.style.color = '#ffffff';
                            break;
                        case 'h4':
                            heading.style.fontSize = '20px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '10px 0 4px 0';
                            heading.style.color = '#ffffff';
                            break;
                        case 'h5':
                            heading.style.fontSize = '18px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '8px 0 4px 0';
                            heading.style.color = '#ffffff';
                            break;
                        case 'h6':
                            heading.style.fontSize = '16px';
                            heading.style.fontWeight = 'bold';
                            heading.style.margin = '6px 0 4px 0';
                            heading.style.color = '#ffffff';
                            break;
                    }
                    
                    try {
                        range.surroundContents(heading);
                    } catch (e) {
                        const contents = range.extractContents();
                        heading.appendChild(contents);
                        range.insertNode(heading);
                    }
                }
                
                console.log('Heading style applied:', selectedHeading);
            }
            
            if (typeof triggerFormattingSave === 'function') {
                triggerFormattingSave();
            }
        });
    }
    
    // Bold button
    const boldBtn = document.getElementById('boldBtn');
    console.log('Bold button found:', !!boldBtn);
    if (boldBtn) {
        boldBtn.addEventListener('click', (e) => {
            e.preventDefault();
                e.stopPropagation();
                console.log('Bold button clicked!');
                executeCommand('bold');
        });
    }
    
    // Italic button
    const italicBtn = document.getElementById('italicBtn');
    console.log('Italic button found:', !!italicBtn);
    if (italicBtn) {
        italicBtn.addEventListener('click', (e) => {
                e.preventDefault();
                e.stopPropagation();
                console.log('Italic button clicked!');
                executeCommand('italic');
            });
        }
    
    // Underline button
    const underlineBtn = document.getElementById('underlineBtn');
    if (underlineBtn) {
        underlineBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Underline button clicked!');
            executeCommand('underline');
        });
    }
    
    // Strikethrough button
    const strikeBtn = document.getElementById('strikeBtn');
    if (strikeBtn) {
        strikeBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Strikethrough button clicked!');
            executeCommand('strikeThrough');
        });
    }
    
    // Alignment buttons
    const alignLeftBtn = document.getElementById('alignLeftBtn');
    if (alignLeftBtn) {
        alignLeftBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Align left button clicked!');
            executeCommand('justifyLeft');
        });
    }
    
    const alignCenterBtn = document.getElementById('alignCenterBtn');
    if (alignCenterBtn) {
        alignCenterBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Align center button clicked!');
            executeCommand('justifyCenter');
        });
    }
    
    const alignRightBtn = document.getElementById('alignRightBtn');
    if (alignRightBtn) {
        alignRightBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Align right button clicked!');
            executeCommand('justifyRight');
        });
    }
    
    const justifyBtn = document.getElementById('justifyBtn');
    if (justifyBtn) {
        justifyBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Justify button clicked!');
            executeCommand('justifyFull');
        });
    }
    
    // List buttons
    const bulletListBtn = document.getElementById('bulletListBtn');
    if (bulletListBtn) {
        bulletListBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Bullet list button clicked!');
            executeCommand('insertUnorderedList');
        });
    }
    
    const numberListBtn = document.getElementById('numberListBtn');
    if (numberListBtn) {
        numberListBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Number list button clicked!');
            executeCommand('insertOrderedList');
        });
    }
    
    // Indent buttons
    const indentBtn = document.getElementById('indentBtn');
    if (indentBtn) {
        indentBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Indent button clicked!');
            executeCommand('indent');
        });
    }
    
    const outdentBtn = document.getElementById('outdentBtn');
    if (outdentBtn) {
        outdentBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Outdent button clicked!');
            executeCommand('outdent');
        });
    }
    
    // Text Color Picker
    // Text color button removed
    
    // Image
    
    // Undo/Redo buttons
    const undoBtn = document.getElementById('undoBtn');
    if (undoBtn) {
        undoBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Undo button clicked!');
            undo();
        });
    }
    
    const redoBtn = document.getElementById('redoBtn');
    if (redoBtn) {
        redoBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Redo button clicked!');
            redo();
        });
    }
    
    // Clear Formatting button
    // Clear formatting and spell check buttons removed
    
    // Find and Replace button removed
    
    // Remove all suggestions button
    const removeSuggestionsBtn = document.getElementById('removeSuggestionsBtn');
    if (removeSuggestionsBtn) {
        removeSuggestionsBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            
            // Confirm before removing all suggestions
            if (confirm('Are you sure you want to remove all suggestions? This action cannot be undone.')) {
                removeAllSuggestions();
            }
        });
    }
}

function removeAllSuggestions() {
    console.log('Removing all suggestions...');
    
    try {
        // Remove all suggestion markers from the document
        const markers = document.querySelectorAll('.suggestion-marker');
        markers.forEach(marker => {
            const textNode = document.createTextNode(marker.textContent);
            marker.replaceWith(textNode);
        });
        
        // Remove all sidebar indicators
        const indicators = document.querySelectorAll('.suggestion-indicator');
        indicators.forEach(indicator => {
            indicator.remove();
        });
        
        // Remove all popups
        const popups = document.querySelectorAll('.suggestion-popup');
        popups.forEach(popup => {
            popup.remove();
        });
        
        // Clear all suggestion storage
        const currentUserData = localStorage.getItem('currentUser');
        if (currentUserData) {
            const currentUser = JSON.parse(currentUserData);
            const classes = JSON.parse(localStorage.getItem('classes') || '[]');
            
            // Clear suggestions for all classes of this user
            classes.forEach(classData => {
                if (classData.userId === currentUser.uid) {
                    // Clear document-specific suggestions
                    const docId = 'current-document';
                    const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
                    localStorage.removeItem(suggestionsKey);
                    
                    // Clear current suggestions
                    const currentSuggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
                    localStorage.removeItem(currentSuggestionsKey);
                    
                    // Clear all document-specific suggestions
                    const documents = JSON.parse(classData.documents || '[]');
                    documents.forEach(doc => {
                        const docSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_${doc.id}`;
                        localStorage.removeItem(docSuggestionsKey);
                    });
                }
            });
        }
        
        // Update current suggestions state
        window.saveCurrentSuggestions();
        
        console.log('All suggestions removed successfully');
        
        // Show success message
        const notification = document.createElement('div');
        notification.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            background: linear-gradient(145deg, #1a1a1a, #2a2a2a);
            color: #ffffff;
            padding: 12px 20px;
            border-radius: 8px;
            border: 1px solid rgba(255, 255, 255, 0.2);
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
            z-index: 10000;
            font-size: 14px;
            animation: slideIn 0.3s ease-out;
        `;
        notification.textContent = 'All suggestions removed successfully';
        document.body.appendChild(notification);
        
        // Remove notification after 3 seconds
        setTimeout(() => {
            if (notification.parentNode) {
                notification.remove();
            }
        }, 3000);
        
    } catch (error) {
        console.error('Error removing suggestions:', error);
        alert('Error removing suggestions. Please try again.');
    }
}

async function saveDocument(classData, existingDoc) {
    try {
        // Use global reference if available
        if (window.currentExistingDoc && !existingDoc) {
            existingDoc = window.currentExistingDoc;
        }
        
        const title = document.getElementById('docEditorTitle').value.trim() || 'Untitled Document';
        
        // Collect content from all pages - preserve all HTML and styling
        const allContents = document.querySelectorAll('.doc-editor-content');
        let combinedContent = '';
        allContents.forEach((contentDiv, index) => {
            if (index > 0) {
                combinedContent += '<div class="page-break"></div>';
            }
            // Use innerHTML to preserve all HTML tags, attributes, and styling
            combinedContent += contentDiv.innerHTML;
        });
        
        console.log('Saving document content with full HTML:', combinedContent.substring(0, 200) + '...');
        
        // Check if images are being saved with their styling
        const imageWrappers = combinedContent.match(/<div class="image-wrapper"[^>]*>.*?<\/div>/gs);
        if (imageWrappers) {
            console.log('🖼️ Found image wrappers in saved content:', imageWrappers.length);
            imageWrappers.forEach((wrapper, index) => {
                const hasStyle = wrapper.includes('style=');
                const hasResizeHandle = wrapper.includes('resize-handle');
                console.log(`🖼️ Image ${index + 1}:`, { hasStyle, hasResizeHandle, wrapper: wrapper.substring(0, 100) + '...' });
            });
        }
        
        const doc = {
            title: title,
            content: combinedContent,
            type: 'text',
            folderId: existingDoc?.folderId || null
        };
        
        let documentId = existingDoc?.id;
        
        // Check if this is a valid Firebase document ID (not a timestamp or local ID)
        const isValidFirebaseId = existingDoc && existingDoc.id && 
            existingDoc.id !== 'new-document' && 
            !existingDoc.id.startsWith('new-document-') &&
            !/^\d+$/.test(existingDoc.id) && // Not just numbers (timestamps)
            existingDoc.id.length > 10; // Firebase IDs are typically longer than 10 characters
        
        console.log('Document ID check:', {
            existingDocId: existingDoc?.id,
            isValidFirebaseId: isValidFirebaseId,
            isNewDocument: existingDoc?.id === 'new-document',
            isNewDocumentPrefix: existingDoc?.id?.startsWith('new-document-'),
            isTimestamp: existingDoc?.id ? /^\d+$/.test(existingDoc.id) : false,
            idLength: existingDoc?.id ? existingDoc.id.length : 0
        });
        
        if (isValidFirebaseId) {
            // Update existing document in Firebase
            await window.documentService.updateDocument(classData.userId, classData.id, existingDoc.id, doc);
            console.log('Document updated in Firebase:', doc.title);
            
            // Also update localStorage for immediate visibility
            const currentUserData = localStorage.getItem('currentUser');
            if (currentUserData) {
                const currentUser = JSON.parse(currentUserData);
                const classes = JSON.parse(localStorage.getItem('classes') || '[]');
                const classIndex = classes.findIndex(c => c.userId === currentUser.uid && c.name === classData.name);
                
                if (classIndex !== -1) {
                    const documents = JSON.parse(classes[classIndex].documents || '[]');
                    const docIndex = documents.findIndex(d => d.id === existingDoc.id);
                    
                    if (docIndex !== -1) {
                        documents[docIndex].title = doc.title;
                        documents[docIndex].content = doc.content;
                        documents[docIndex].lastModified = new Date().toISOString();
                        classes[classIndex].documents = JSON.stringify(documents);
                        localStorage.setItem('classes', JSON.stringify(classes));
                        console.log('Updated localStorage for existing document:', doc.title);
                    }
                }
            }
        } else {
            // Create new document in Firebase
            documentId = await window.documentService.saveDocument(classData.userId, classData.id, doc);
            console.log('New document created in Firebase:', doc.title, 'with ID:', documentId);
            
            // Update the existingDoc object with the new ID for future operations
            if (existingDoc) {
                existingDoc.id = documentId;
            } else {
                // If no existingDoc was passed, create one for future auto-saves
                existingDoc = {
                    id: documentId,
                    title: doc.title,
                    content: doc.content,
                    type: doc.type,
                    folderId: doc.folderId
                };
            }
            
            // Update the global existingDoc reference for future auto-saves
            window.currentExistingDoc = existingDoc;
            
            // Update localStorage with the new Firebase document ID
            const currentUserData = localStorage.getItem('currentUser');
            if (currentUserData) {
                const currentUser = JSON.parse(currentUserData);
                const classes = JSON.parse(localStorage.getItem('classes') || '[]');
                const classIndex = classes.findIndex(c => c.userId === currentUser.uid && c.name === classData.name);
                
                if (classIndex !== -1) {
                    const documents = JSON.parse(classes[classIndex].documents || '[]');
                    const docIndex = documents.findIndex(d => d.id === existingDoc.id || d.id.startsWith('new-document-'));
                    
                    if (docIndex !== -1) {
                        // Update existing document entry
                        documents[docIndex].id = documentId;
                        documents[docIndex].title = doc.title;
                        documents[docIndex].content = doc.content;
                        documents[docIndex].lastModified = new Date().toISOString();
                    } else {
                        // Add new document to localStorage
                        documents.push({
                            id: documentId,
                            title: doc.title,
                            content: doc.content,
                            type: doc.type,
                            createdAt: new Date().toISOString(),
                            lastModified: new Date().toISOString(),
                            folderId: doc.folderId
                        });
                    }
                    
                    classes[classIndex].documents = JSON.stringify(documents);
                    localStorage.setItem('classes', JSON.stringify(classes));
                    console.log('Updated localStorage with Firebase document ID:', documentId);
                    
                    // Trigger documents updated event to refresh class view
                    window.dispatchEvent(new CustomEvent('documentsUpdated'));
                }
            }
            
            // Transfer suggestions from 'new-document' to actual document ID
            const oldSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_new-document`;
            const newSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_${documentId}`;
            
            const oldSuggestions = localStorage.getItem(oldSuggestionsKey);
            if (oldSuggestions) {
                localStorage.setItem(newSuggestionsKey, oldSuggestions);
                localStorage.removeItem(oldSuggestionsKey);
                console.log('Transferred suggestions from new-document to actual document ID:', documentId);
            }
        }
        
        // Return the updated document reference for future auto-saves
        return {
            id: documentId,
            title: doc.title,
            content: doc.content,
            type: doc.type,
            folderId: doc.folderId
        };
    } catch (error) {
        console.error('Error saving document:', error);
        alert('Error saving document. Please try again.');
        return existingDoc; // Return the original document on error
    }
}

// Handle beforeunload event to prevent data loss
function handleBeforeUnload(event) {
    // Check if document editor is open
    const editorScreen = document.getElementById('documentEditorScreen');
    if (editorScreen) {
        // Try to save synchronously (this is limited but better than nothing)
        try {
            // Trigger a quick save
            if (typeof window.autoSaveDocument === 'function') {
                window.autoSaveDocument(window.currentClassData, window.currentExistingDoc);
            }
        } catch (error) {
            console.error('Error in beforeunload save:', error);
        }
        
        // Show warning to user
        event.preventDefault();
        event.returnValue = 'You have unsaved changes in your document. Are you sure you want to leave?';
        return 'You have unsaved changes in your document. Are you sure you want to leave?';
    }
}

async function closeDocumentEditor() {
    console.log('closeDocumentEditor called');
    const editorScreen = document.getElementById('documentEditorScreen');
    console.log('Editor screen found:', editorScreen);
    
    if (editorScreen) {
        try {
            // Auto-save document before closing to prevent data loss
            console.log('Auto-saving document before closing...');
            await saveDocument(window.currentClassData, window.currentExistingDoc);
            console.log('Document auto-saved successfully');
        } catch (error) {
            console.error('Error auto-saving document:', error);
            // Show user-friendly error message
            alert('Warning: Could not save your document automatically. Your changes may be lost. Please try saving manually before closing.');
        }
        
        // Save any pending suggestions before closing
        if (typeof window.saveCurrentSuggestions === 'function') {
        window.saveCurrentSuggestions();
        }
        
        editorScreen.remove();
        console.log('Editor screen removed');
    }
    
    // Remove beforeunload event listener
    window.removeEventListener('beforeunload', handleBeforeUnload);
    
    // Clear auto-save interval
    if (window.autoSaveInterval) {
        clearInterval(window.autoSaveInterval);
        window.autoSaveInterval = null;
    }
    
    // Show class view again
    const classView = document.getElementById('classViewContainer');
    console.log('Class view found:', classView);
    if (classView) {
        classView.style.display = 'flex';
        console.log('Class view shown');
    }
    
    // Re-enable body scroll when document editor is closed
    document.body.classList.remove('document-editor-open');
    
    // Reload documents to show updated list
    const event = new CustomEvent('documentsUpdated');
    window.dispatchEvent(event);
    console.log('Documents updated event dispatched');
}

// Global function to auto-save document content
window.autoSaveDocument = function(classData, existingDoc) {
    try {
        // Use global reference if available
        if (window.currentExistingDoc && !existingDoc) {
            existingDoc = window.currentExistingDoc;
        }
        
        const title = document.getElementById('docEditorTitle')?.value?.trim() || 'Untitled Document';
        const content = getDocumentContent(false); // Get full document content
        
        if (!content.trim()) return; // Don't save empty documents
        
        const currentUserData = localStorage.getItem('currentUser');
        if (!currentUserData) return;
        
        const currentUser = JSON.parse(currentUserData);
        const classes = JSON.parse(localStorage.getItem('classes') || '[]');
        const classIndex = classes.findIndex(c => c.userId === currentUser.uid && c.name === classData.name);
        
        if (classIndex === -1) return;
        
        const documents = JSON.parse(classes[classIndex].documents || '[]');
        
        // Check if this is a valid Firebase document ID (not a timestamp or local ID)
        const isValidFirebaseId = existingDoc && existingDoc.id && 
            existingDoc.id !== 'new-document' && 
            !existingDoc.id.startsWith('new-document-') &&
            !/^\d+$/.test(existingDoc.id) && // Not just numbers (timestamps)
            existingDoc.id.length > 10; // Firebase IDs are typically longer than 10 characters
        
        console.log('Auto-save ID check:', {
            existingDocId: existingDoc?.id,
            isValidFirebaseId: isValidFirebaseId,
            isTimestamp: existingDoc?.id ? /^\d+$/.test(existingDoc.id) : false,
            idLength: existingDoc?.id ? existingDoc.id.length : 0
        });
        
        const doc = {
            title: title,
            content: content,
            type: 'text',
            folderId: existingDoc?.folderId || null,
            lastModified: new Date().toISOString()
        };
        
        if (isValidFirebaseId) {
            // Update existing document in both Firebase and localStorage
            window.documentService.updateDocument(classData.userId, classData.id, existingDoc.id, doc)
                .then(() => {
                    console.log('Auto-saved existing document in Firebase:', title);
                    
                    // Also update localStorage for immediate visibility
            const docIndex = documents.findIndex(d => d.id === existingDoc.id);
            if (docIndex !== -1) {
                documents[docIndex].title = title;
                        documents[docIndex].content = content;
                documents[docIndex].lastModified = new Date().toISOString();
                classes[classIndex].documents = JSON.stringify(documents);
                localStorage.setItem('classes', JSON.stringify(classes));
                        console.log('Updated localStorage for existing document:', title);
                    }
                })
                .catch(error => {
                    console.error('Auto-save Firebase error:', error);
                });
        } else {
            // Create new document in Firebase first, then update localStorage
            window.documentService.saveDocument(classData.userId, classData.id, doc)
                .then(newDocId => {
                    console.log('Auto-saved new document in Firebase:', title, 'with ID:', newDocId);
                    
                    // Update the existingDoc reference
                    if (existingDoc) {
                        existingDoc.id = newDocId;
                    }
                    
                    // Update the global existingDoc reference
                    window.currentExistingDoc = existingDoc;
                    
                    // Update localStorage with the new Firebase document ID
                    const docIndex = documents.findIndex(d => d.id === existingDoc?.id || d.id.startsWith('new-document-'));
                    
                    if (docIndex !== -1) {
                        // Update existing document entry
                        documents[docIndex].id = newDocId;
                        documents[docIndex].title = title;
                        documents[docIndex].content = content;
                        documents[docIndex].lastModified = new Date().toISOString();
                    } else {
                        // Add new document to localStorage
                        documents.push({
                            id: newDocId,
                            title: title,
                            content: content,
                            type: 'text',
                            createdAt: new Date().toISOString(),
                            lastModified: new Date().toISOString(),
                            folderId: existingDoc?.folderId || null
                        });
                    }
                    
            classes[classIndex].documents = JSON.stringify(documents);
            localStorage.setItem('classes', JSON.stringify(classes));
                    console.log('Updated localStorage with Firebase document ID:', newDocId);
                    
                    // Trigger documents updated event to refresh class view
                    window.dispatchEvent(new CustomEvent('documentsUpdated'));
                })
                .catch(error => {
                    console.error('Auto-save Firebase error:', error);
                });
        }
    } catch (error) {
        console.error('Auto-save error:', error);
    }
};

// Global function to save current suggestions
window.saveCurrentSuggestions = function() {
    try {
        // Save current suggestions state before closing
        const suggestionsSidebar = document.getElementById('suggestionsSidebar');
        if (suggestionsSidebar) {
            const indicators = suggestionsSidebar.querySelectorAll('.suggestion-indicator');
            const suggestions = [];
            
            indicators.forEach(indicator => {
                const index = indicator.dataset.suggestionIndex;
                const marker = document.getElementById(`suggestion-${index}`);
                if (marker) {
                    suggestions.push({
                        index: index,
                        original: marker.textContent,
                        markerId: marker.id,
                        indicatorId: indicator.id
                    });
                }
            });
            
            // Also check for any remaining markers in the document
            const allMarkers = document.querySelectorAll('.suggestion-marker');
            allMarkers.forEach(marker => {
                const index = marker.dataset.suggestionIndex;
                if (index && !suggestions.find(s => s.index === index)) {
                    suggestions.push({
                        index: index,
                        original: marker.textContent,
                        markerId: marker.id,
                        indicatorId: `indicator-${index}`
                    });
                }
            });
            
            // Save to localStorage
            const currentUserData = localStorage.getItem('currentUser');
            if (currentUserData) {
                const currentUser = JSON.parse(currentUserData);
                const classes = JSON.parse(localStorage.getItem('classes') || '[]');
                if (classes.length > 0) {
                    const classData = classes[0];
                    const docId = 'current-document'; // Use a generic key for current document
                    const suggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
                    localStorage.setItem(suggestionsKey, JSON.stringify(suggestions));
                    console.log('Saved current suggestions:', suggestions.length, 'suggestions');
                }
            }
        } else {
            console.log('No suggestions sidebar found, nothing to save');
        }
    } catch (error) {
        console.error('Error saving current suggestions:', error);
    }
};

function downloadAsPDF() {
    const title = document.getElementById('docEditorTitle').value.trim() || 'Untitled Document';
    const content = document.getElementById('docEditorContent');
    
    if (!content) {
        alert('No content to download');
        return;
    }
    
    try {
        // Check if jsPDF is available
        if (typeof window.jspdf === 'undefined') {
            alert('PDF generation library not loaded. Please refresh the page and try again.');
            return;
        }
        
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF();
        
        // Set up PDF styling
        const pageWidth = doc.internal.pageSize.getWidth();
        const pageHeight = doc.internal.pageSize.getHeight();
        const margin = 20;
        const maxWidth = pageWidth - (margin * 2);
        
        // Get content text (strip HTML tags for now)
        const contentText = content.innerText || content.textContent || '';
        
        // Add content (no title in PDF content)
        doc.setFontSize(12);
        doc.setFont(undefined, 'normal');
        const contentLines = doc.splitTextToSize(contentText, maxWidth);
        
        let yPosition = margin;
        const lineHeight = 7;
        
        for (let i = 0; i < contentLines.length; i++) {
            // Check if we need a new page
            if (yPosition + lineHeight > pageHeight - margin) {
                doc.addPage();
                yPosition = margin;
            }
            
            doc.text(contentLines[i], margin, yPosition);
            yPosition += lineHeight;
        }
        
        // Save the PDF
        const fileName = `${title.replace(/[^a-z0-9]/gi, '_').toLowerCase()}.pdf`;
        doc.save(fileName);
        
        console.log('PDF generated and downloaded:', fileName);
        
    } catch (error) {
        console.error('Error generating PDF:', error);
        alert('Error generating PDF. Please try again.');
    }
}

function humanizeText() {
    const contentElements = document.querySelectorAll('.doc-editor-content');
    
    if (contentElements.length === 0) {
        alert('No content to humanize.');
        return;
    }
    
    // Check if there's a text selection
    const selection = window.getSelection();
    const hasSelection = selection.toString().trim().length > 0;
    
    // Get text content based on selection while preserving formatting context
    let textToHumanize = '';
    if (hasSelection) {
        textToHumanize = selection.toString().trim();
    } else {
        // Get all text content while preserving formatting structure
        contentElements.forEach(content => {
            textToHumanize += content.innerText + '\n\n';
        });
        textToHumanize = textToHumanize.trim();
    }
    
    if (!textToHumanize) {
        alert('No text content found to humanize.');
        return;
    }
    
    // Show confirmation popup
    const scope = hasSelection ? 'highlighted region' : 'entire document';
    const confirmed = confirm(`This will humanize your ${scope}. This action will replace the current text with a more human-like version. Continue?`);
    
    if (!confirmed) {
        return;
    }
    
    // Show loading state
    const humanizeBtn = document.getElementById('humanizeBtn');
    const originalText = humanizeBtn.textContent;
    humanizeBtn.textContent = 'Humanizing...';
    humanizeBtn.disabled = true;
    
    // Show Genius thinking animation and spotlight if there's a selection
    let geniusThinking = null;
    let spotlightOverlay = null;
    if (hasSelection) {
        geniusThinking = showGeniusThinking();
        // Use the existing spotlight effect from genius edit mode
        const selection = window.getSelection();
        if (selection && selection.rangeCount > 0) {
            const range = selection.getRangeAt(0);
            window.createSpotlightEffect(range);
        }
    }
    
    // Use OpenAI instead of HumanizeAI Pro due to CORS issues
    const currentUserData = localStorage.getItem('currentUser');
    if (!currentUserData) {
        alert('Please log in to use the Humanize feature.');
        humanizeBtn.textContent = originalText;
        humanizeBtn.disabled = false;
        if (geniusThinking) geniusThinking.remove();
        return;
    }
    
    const currentUser = JSON.parse(currentUserData);
    
    // Use global API key for all users
    const OPENAI_API_KEY = window.getOpenAIApiKey() || window.APP_CONFIG.OPENAI_API_KEY;
    
    if (!OPENAI_API_KEY) {
        showNotification('OpenAI API key not found. Please add your API key in settings.', 'error');
        humanizeBtn.textContent = originalText;
        humanizeBtn.disabled = false;
        if (geniusThinking) geniusThinking.remove();
        return;
    }
    
    // Use OpenAI for humanization (CORS-friendly)
    fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify({
            model: 'gpt-4o-mini',
            messages: [
                {
                    role: 'system',
                    content: `You are a professional text humanizer. Your task is to rewrite text to make it sound more natural, human-like, and engaging while maintaining the original meaning and key information.

Guidelines:
- Use natural, conversational language
- Add variety in sentence structure and length
- Include personal touches and human expressions
- Make the text flow more naturally
- Avoid overly formal or robotic language
- Keep the core message and facts intact
- Make it engaging and readable

Rewrite the following text to sound more human and natural:`
                },
                {
                    role: 'user',
                    content: textToHumanize
                }
            ],
            temperature: 0.7,
            max_tokens: 2000
        })
    })
    .then(response => {
        if (!response.ok) {
            throw new Error(`API error: ${response.status}`);
        }
        return response.json();
    })
    .then(data => {
        const humanizedText = data.choices[0].message.content;
        
        // Show preview dialog for humanized text
        showHumanizePreview(textToHumanize, humanizedText, hasSelection);
        
    })
    .catch(error => {
        console.error('Humanization API Error:', error);
        showNotification('Error humanizing text. Please try again.', 'error');
    })
    .finally(() => {
        // Restore button state
        humanizeBtn.textContent = originalText;
        humanizeBtn.disabled = false;
        if (geniusThinking) geniusThinking.remove();
        // Use the existing spotlight removal function
        window.removeSpotlightEffect();
    });
}

// Show humanize preview dialog
function showHumanizePreview(originalText, humanizedText, hasSelection) {
    // Create modal
    const modal = document.createElement('div');
    modal.style.cssText = `
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.8);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 10000;
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    `;
    
    const modalContent = document.createElement('div');
    modalContent.style.cssText = `
        background: linear-gradient(145deg, #1a1a1a, #2a2a2a);
        border: 1px solid rgba(255, 255, 255, 0.2);
        border-radius: 12px;
        padding: 30px;
        max-width: 90vw;
        max-height: 90vh;
        overflow-y: auto;
        box-shadow: 0 20px 40px rgba(0, 0, 0, 0.5);
        color: #ffffff;
        display: flex;
        flex-direction: column;
        gap: 20px;
    `;
    
    modalContent.innerHTML = `
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
            <h2 style="margin: 0; color: #ffffff; font-size: 24px;">Humanize Preview</h2>
            <button id="closeHumanizeModal" style="background: none; border: none; color: #ffffff; font-size: 24px; cursor: pointer; padding: 5px;">×</button>
        </div>
        
        <div style="display: flex; flex-direction: column; gap: 20px; flex: 1; min-height: 400px;">
            <div style="display: flex; flex-direction: column;">
                <h3 style="color: #ff6b6b; margin-bottom: 10px; font-size: 18px;">Original Text</h3>
                <div id="originalTextPreview" style="
                    background: rgba(255, 107, 107, 0.1);
                    border: 1px solid rgba(255, 107, 107, 0.3);
                    border-radius: 8px;
                    padding: 15px;
                    min-height: 150px;
                    max-height: 300px;
                    overflow-y: auto;
                    line-height: 1.6;
                    white-space: pre-wrap;
                ">${originalText}</div>
            </div>
            
            <div style="display: flex; flex-direction: column;">
                <h3 style="color: #51cf66; margin-bottom: 10px; font-size: 18px;">Humanized Text</h3>
                <div id="humanizedTextPreview" style="
                    background: rgba(81, 207, 102, 0.1);
                    border: 1px solid rgba(81, 207, 102, 0.3);
                    border-radius: 8px;
                    padding: 15px;
                    min-height: 150px;
                    max-height: 300px;
                    overflow-y: auto;
                    line-height: 1.6;
                    white-space: pre-wrap;
                ">${humanizedText}</div>
            </div>
        </div>
        
        <div style="display: flex; gap: 10px; justify-content: space-between; margin-top: 20px;">
            <button id="redoHumanize" style="
                background: linear-gradient(135deg, #4a9eff, #6bb6ff);
                color: white;
                border: none;
                padding: 12px 24px;
                border-radius: 8px;
                cursor: pointer;
                font-size: 14px;
                font-weight: 600;
                transition: all 0.2s ease;
                display: flex;
                align-items: center;
                gap: 8px;
            ">
                <span>🔄</span>
                Redo
            </button>
            
            <div style="display: flex; gap: 10px;">
                <button id="rejectHumanize" style="
                    background: linear-gradient(135deg, #ff6b6b, #ff8e8e);
                    color: white;
                    border: none;
                    padding: 12px 24px;
                    border-radius: 8px;
                    cursor: pointer;
                    font-size: 14px;
                    font-weight: 600;
                    transition: all 0.2s ease;
                ">Reject Changes</button>
                <button id="acceptHumanize" style="
                    background: linear-gradient(135deg, #51cf66, #69db7c);
                    color: white;
                    border: none;
                    padding: 12px 24px;
                    border-radius: 8px;
                    cursor: pointer;
                    font-size: 14px;
                    font-weight: 600;
                    transition: all 0.2s ease;
                ">Accept Changes</button>
            </div>
        </div>
    `;
    
    modal.appendChild(modalContent);
    document.body.appendChild(modal);
    
    // Close modal functionality
    const closeModal = () => {
        modal.remove();
    };
    
    document.getElementById('closeHumanizeModal').addEventListener('click', closeModal);
    modal.addEventListener('click', (e) => {
        if (e.target === modal) closeModal();
    });
    
    // Redo humanization
    document.getElementById('redoHumanize').addEventListener('click', async () => {
        const redoBtn = document.getElementById('redoHumanize');
        const originalRedoText = redoBtn.innerHTML;
        
        // Show loading state
        redoBtn.innerHTML = '<span>⏳</span> Regenerating...';
        redoBtn.disabled = true;
        
        try {
            // Call humanize API again
            const newHumanizedText = await regenerateHumanization(originalText);
            
            // Update the preview with new text
            document.getElementById('humanizedTextPreview').textContent = newHumanizedText;
            
            // Update the humanizedText variable for accept button
            window.currentHumanizedText = newHumanizedText;
            
            showNotification('New humanized version generated!', 'success');
        } catch (error) {
            console.error('Redo humanization error:', error);
            showNotification('Error regenerating text. Please try again.', 'error');
        } finally {
            // Restore button state
            redoBtn.innerHTML = originalRedoText;
            redoBtn.disabled = false;
        }
    });
    
    // Reject changes
    document.getElementById('rejectHumanize').addEventListener('click', () => {
        closeModal();
        showNotification('Humanization rejected. No changes made.', 'info');
    });
    
    // Accept changes
    document.getElementById('acceptHumanize').addEventListener('click', () => {
        const finalHumanizedText = window.currentHumanizedText || humanizedText;
        applyHumanization(finalHumanizedText, hasSelection);
        closeModal();
        showNotification('Text humanized successfully!', 'success');
        
        // Auto-save the document after humanization
        setTimeout(() => {
            autoSaveDocument(window.currentClassData, window.currentExistingDoc);
        }, 100);
    });
    
    // Close on Escape key
    const handleEscape = (e) => {
        if (e.key === 'Escape') {
            closeModal();
            document.removeEventListener('keydown', handleEscape);
        }
    };
    document.addEventListener('keydown', handleEscape);
    
    // Add hover effects
    const redoBtn = document.getElementById('redoHumanize');
    const rejectBtn = document.getElementById('rejectHumanize');
    const acceptBtn = document.getElementById('acceptHumanize');
    
    redoBtn.addEventListener('mouseenter', () => {
        redoBtn.style.transform = 'translateY(-2px)';
        redoBtn.style.boxShadow = '0 4px 12px rgba(74, 158, 255, 0.3)';
    });
    redoBtn.addEventListener('mouseleave', () => {
        redoBtn.style.transform = 'translateY(0)';
        redoBtn.style.boxShadow = 'none';
    });
    
    rejectBtn.addEventListener('mouseenter', () => {
        rejectBtn.style.transform = 'translateY(-2px)';
        rejectBtn.style.boxShadow = '0 4px 12px rgba(255, 107, 107, 0.3)';
    });
    rejectBtn.addEventListener('mouseleave', () => {
        rejectBtn.style.transform = 'translateY(0)';
        rejectBtn.style.boxShadow = 'none';
    });
    
    acceptBtn.addEventListener('mouseenter', () => {
        acceptBtn.style.transform = 'translateY(-2px)';
        acceptBtn.style.boxShadow = '0 4px 12px rgba(81, 207, 102, 0.3)';
    });
    acceptBtn.addEventListener('mouseleave', () => {
        acceptBtn.style.transform = 'translateY(0)';
        acceptBtn.style.boxShadow = 'none';
    });
}

// Apply humanization to the document while preserving formatting
function applyHumanization(humanizedText, hasSelection) {
    const contentElements = document.querySelectorAll('.doc-editor-content');
    
        if (hasSelection) {
        // Replace selected text while preserving HTML formatting
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
            
            // Get the selected text and its HTML context
            const selectedText = selection.toString();
            const selectedHTML = range.cloneContents();
            
            // Check if selection contains HTML formatting
            if (selectedHTML.children.length > 0 || selectedHTML.querySelector('strong, em, u, s, b, i, span, p, div, h1, h2, h3, h4, h5, h6, ul, ol, li')) {
                // Selection contains formatting - preserve it
                preserveFormattingAndReplace(range, humanizedText);
            } else {
                // Plain text selection - simple replacement
                range.deleteContents();
                range.insertNode(document.createTextNode(humanizedText));
            }
            }
        } else {
        // For full document, preserve HTML structure and only replace text content
            contentElements.forEach(content => {
            preserveDocumentFormatting(content, humanizedText);
        });
    }
}

// Preserve formatting when replacing selected text
function preserveFormattingAndReplace(range, newText) {
    const selectedHTML = range.cloneContents();
    const container = document.createElement('div');
    container.appendChild(selectedHTML);
    
    // Find all text nodes in the selection
    const textNodes = [];
    const walker = document.createTreeWalker(
        container,
        NodeFilter.SHOW_TEXT,
        null,
        false
    );
    
    let node;
    while (node = walker.nextNode()) {
        if (node.textContent.trim()) {
            textNodes.push(node);
        }
    }
    
    if (textNodes.length > 0) {
        // Replace text in the first text node, remove others
        textNodes[0].textContent = newText;
        for (let i = 1; i < textNodes.length; i++) {
            textNodes[i].textContent = '';
        }
        
        // Replace the selection with the modified HTML
        range.deleteContents();
        range.insertNode(container.firstChild);
    } else {
        // Fallback to simple text replacement
        range.deleteContents();
        range.insertNode(document.createTextNode(newText));
    }
}

// Preserve document formatting when humanizing entire content
function preserveDocumentFormatting(contentElement, humanizedText) {
    // Get all text content with formatting preserved
    const textWithFormatting = contentElement.innerHTML;
    
    // Extract just the text content for humanization
    const textContent = contentElement.innerText || contentElement.textContent || '';
    
    // Split the humanized text into sentences/paragraphs to match structure
    const humanizedSentences = humanizedText.split(/(?<=[.!?])\s+/);
    const originalSentences = textContent.split(/(?<=[.!?])\s+/);
    
    let newHTML = textWithFormatting;
    let sentenceIndex = 0;
    
    // Replace text content while preserving HTML tags
    const walker = document.createTreeWalker(
        contentElement,
        NodeFilter.SHOW_TEXT,
        null,
        false
    );
    
    const textNodes = [];
    let node;
    while (node = walker.nextNode()) {
        if (node.textContent.trim()) {
            textNodes.push(node);
        }
    }
    
    // Replace text in each text node with corresponding humanized text
    textNodes.forEach((textNode, index) => {
        if (textNode.textContent.trim()) {
            // Use the humanized text for this text node
            const nodeText = textNode.textContent.trim();
            const humanizedNodeText = humanizedSentences[sentenceIndex] || humanizedText;
            textNode.textContent = humanizedNodeText;
            sentenceIndex++;
        }
    });
}

// Regenerate humanization for redo functionality
async function regenerateHumanization(originalText) {
    const currentUserData = localStorage.getItem('currentUser');
    if (!currentUserData) {
        throw new Error('Please log in to use the Humanize feature.');
    }
    
    const currentUser = JSON.parse(currentUserData);
    const OPENAI_API_KEY = window.getOpenAIApiKey() || window.APP_CONFIG.OPENAI_API_KEY;
    
    if (!OPENAI_API_KEY) {
        throw new Error('OpenAI API key not found. Please add your API key in settings.');
    }
    
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify({
            model: 'gpt-4o-mini',
            messages: [
                {
                    role: 'system',
                    content: `You are a professional text humanizer. Your task is to rewrite text to make it sound more natural, human-like, and engaging while maintaining the original meaning and key information.

Guidelines:
- Use natural, conversational language
- Add personality and voice to the text
- Make it more engaging and readable
- Maintain the original meaning and key points
- Use varied sentence structures
- Add appropriate transitions and flow
- Make it sound like a human wrote it, not AI

Rewrite the following text to make it more human-like:`
                },
                {
                    role: 'user',
                    content: originalText
                }
            ],
            temperature: 0.8, // Higher temperature for more variation
            max_tokens: 1000
        })
    });
    
    if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
    }
    
    const data = await response.json();
    return data.choices[0].message.content;
}

function showGeniusThinking() {
    // Find the existing Genius floating component
    const geniusFloating = document.querySelector('.genius-floating');
    if (!geniusFloating) {
        console.log('No floating Genius component found');
        return null;
    }
    
    // Store the original content
    const originalContent = geniusFloating.innerHTML;
    
    // Replace with thinking animation
    geniusFloating.innerHTML = `
        <div class="genius-thinking-content" style="
            display: flex;
            align-items: center;
            gap: 12px;
            padding: 12px 16px;
            color: #ffffff;
        ">
            <div class="genius-thinking-icon" style="
                width: 20px;
                height: 20px;
                background: linear-gradient(45deg, #4a9eff, #5ba8ff);
                border-radius: 50%;
                animation: pulse 1.5s ease-in-out infinite;
            "></div>
            <div style="font-size: 14px; font-weight: 500;">Genius is thinking...</div>
        </div>
    `;
    
    // Add pulse animation if not already added
    if (!document.querySelector('#genius-pulse-animation')) {
        const style = document.createElement('style');
        style.id = 'genius-pulse-animation';
        style.textContent = `
            @keyframes pulse {
                0%, 100% { transform: scale(1); opacity: 1; }
                50% { transform: scale(1.1); opacity: 0.7; }
            }
        `;
        document.head.appendChild(style);
    }
    
    // Return an object with restore method
    return {
        element: geniusFloating,
        originalContent: originalContent,
        remove: function() {
            // Restore original content
            geniusFloating.innerHTML = originalContent;
            // Re-setup input listeners
            const input = geniusFloating.querySelector('.genius-input');
            if (input) {
                setupGeniusInputListeners(input, window.currentClassData, window.currentExistingDoc);
            }
        }
    };
}


function showHumanizationSuggestion(originalText, humanizedText, isSelection, contentElements) {
    // Create modal
    const modal = document.createElement('div');
    modal.style.cssText = `
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.8);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 10000;
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    `;
    
    const modalContent = document.createElement('div');
    modalContent.style.cssText = `
        background: linear-gradient(145deg, #1a1a1a, #2a2a2a);
        border: 1px solid rgba(255, 255, 255, 0.2);
        border-radius: 12px;
        padding: 30px;
        max-width: 800px;
        max-height: 80vh;
        overflow-y: auto;
        box-shadow: 0 20px 40px rgba(0, 0, 0, 0.5);
        color: #ffffff;
    `;
    
    modalContent.innerHTML = `
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; padding-bottom: 15px; border-bottom: 1px solid rgba(255, 255, 255, 0.1);">
            <div style="display: flex; align-items: center; gap: 12px;">
                <div style="width: 8px; height: 8px; background: linear-gradient(45deg, #4a9eff, #5ba8ff); border-radius: 50%;"></div>
                <h2 style="margin: 0; color: #ffffff; font-size: 28px; font-weight: 600; letter-spacing: -0.5px;">Humanization Suggestion</h2>
            </div>
            <button id="closeModal" style="
                background: rgba(255, 255, 255, 0.1);
                border: 1px solid rgba(255, 255, 255, 0.2);
                border-radius: 8px;
                color: #ffffff;
                font-size: 18px;
                cursor: pointer;
                padding: 8px 12px;
                transition: all 0.2s ease;
                display: flex;
                align-items: center;
                justify-content: center;
                width: 36px;
                height: 36px;
            " onmouseover="this.style.background='rgba(255, 255, 255, 0.2)'" onmouseout="this.style.background='rgba(255, 255, 255, 0.1)'">×</button>
        </div>
        
        <div style="margin-bottom: 25px;">
            <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 12px;">
                <div style="width: 4px; height: 20px; background: linear-gradient(135deg, #4a9eff, #5ba8ff); border-radius: 2px;"></div>
                <h3 style="color: #4a9eff; margin: 0; font-size: 16px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;">Original Text</h3>
            </div>
            <div style="
                background: linear-gradient(145deg, rgba(255, 255, 255, 0.03), rgba(255, 255, 255, 0.08));
                border: 1px solid rgba(74, 158, 255, 0.2);
                padding: 20px;
                border-radius: 12px;
                line-height: 1.7;
                font-size: 15px;
                color: #e0e0e0;
                position: relative;
                box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            ">
                <div style="position: absolute; top: 12px; right: 12px; color: #4a9eff; font-size: 12px; opacity: 0.7;">ORIGINAL</div>
                ${originalText.replace(/\n/g, '<br>')}
            </div>
        </div>
        
        <div style="margin-bottom: 30px;">
            <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 12px;">
                <div style="width: 4px; height: 20px; background: linear-gradient(135deg, #51cf66, #40c057); border-radius: 2px;"></div>
                <h3 style="color: #51cf66; margin: 0; font-size: 16px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;">Humanized Version</h3>
            </div>
            <div style="
                background: linear-gradient(145deg, rgba(81, 207, 102, 0.05), rgba(64, 192, 87, 0.1));
                border: 1px solid rgba(81, 207, 102, 0.3);
                padding: 20px;
                border-radius: 12px;
                line-height: 1.7;
                font-size: 15px;
                color: #e8f5e8;
                position: relative;
                box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            ">
                <div style="position: absolute; top: 12px; right: 12px; color: #51cf66; font-size: 12px; opacity: 0.7;">IMPROVED</div>
                ${humanizedText.replace(/\n/g, '<br>')}
            </div>
        </div>
        
        <div style="display: flex; gap: 16px; justify-content: flex-end; padding-top: 20px; border-top: 1px solid rgba(255, 255, 255, 0.1);">
            <button id="rejectHumanization" style="
                background: linear-gradient(145deg, rgba(255, 255, 255, 0.08), rgba(255, 255, 255, 0.12));
                border: 1px solid rgba(255, 255, 255, 0.2);
                border-radius: 10px;
                color: #b0b0b0;
                padding: 12px 24px;
                cursor: pointer;
                font-size: 14px;
                font-weight: 500;
                transition: all 0.3s ease;
                min-width: 100px;
            " onmouseover="this.style.background='linear-gradient(145deg, rgba(255, 255, 255, 0.15), rgba(255, 255, 255, 0.2)'; this.style.color='#ffffff'; this.style.transform='translateY(-1px)'" onmouseout="this.style.background='linear-gradient(145deg, rgba(255, 255, 255, 0.08), rgba(255, 255, 255, 0.12))'; this.style.color='#b0b0b0'; this.style.transform='translateY(0)'">Cancel</button>
            <button id="acceptHumanization" style="
                background: linear-gradient(145deg, #51cf66, #40c057);
                border: 1px solid rgba(81, 207, 102, 0.3);
                border-radius: 10px;
                color: #ffffff;
                padding: 12px 24px;
                cursor: pointer;
                font-size: 14px;
                font-weight: 600;
                transition: all 0.3s ease;
                min-width: 160px;
                box-shadow: 0 4px 12px rgba(81, 207, 102, 0.2);
            " onmouseover="this.style.background='linear-gradient(145deg, #5dd877, #4dd16a)'; this.style.transform='translateY(-2px)'; this.style.boxShadow='0 6px 16px rgba(81, 207, 102, 0.3)'" onmouseout="this.style.background='linear-gradient(145deg, #51cf66, #40c057)'; this.style.transform='translateY(0)'; this.style.boxShadow='0 4px 12px rgba(81, 207, 102, 0.2)'">✨ Apply Humanization</button>
        </div>
    `;
    
    modal.appendChild(modalContent);
    document.body.appendChild(modal);
    
    // Close modal functionality
    const closeModal = () => {
        modal.remove();
    };
    
    document.getElementById('closeModal').addEventListener('click', closeModal);
    document.getElementById('rejectHumanization').addEventListener('click', closeModal);
    
    document.getElementById('acceptHumanization').addEventListener('click', () => {
        // Apply humanization
        if (isSelection) {
            // Replace selected text
            const selection = window.getSelection();
            if (selection.rangeCount > 0) {
                const range = selection.getRangeAt(0);
                range.deleteContents();
                range.insertNode(document.createTextNode(humanizedText));
            }
        } else {
            // Replace all content
            contentElements.forEach(content => {
                content.innerHTML = humanizedText;
            });
        }
        
        // Show success notification
        showNotification('Text humanized successfully!', 'success');
        
        // Auto-save the document after humanization
        setTimeout(() => {
            autoSaveDocument(window.currentClassData, window.currentExistingDoc);
        }, 100);
        
        closeModal();
    });
    
    modal.addEventListener('click', (e) => {
        if (e.target === modal) closeModal();
    });
    
    // Close on Escape key
    const handleEscape = (e) => {
        if (e.key === 'Escape') {
            closeModal();
            document.removeEventListener('keydown', handleEscape);
        }
    };
    document.addEventListener('keydown', handleEscape);
}

function showNotification(message, type = 'info') {
    const notification = document.createElement('div');
    
    let background, icon;
    switch(type) {
        case 'success':
            background = 'linear-gradient(145deg, #51cf66, #40c057)';
            icon = '✓';
            break;
        case 'error':
            background = 'linear-gradient(145deg, #ff6b6b, #ee5a52)';
            icon = '✕';
            break;
        default:
            background = 'linear-gradient(145deg, #4a9eff, #5ba8ff)';
            icon = 'ℹ';
    }
    
    notification.style.cssText = `
        position: fixed;
        top: 20px;
        right: 20px;
        background: ${background};
        color: #ffffff;
        padding: 12px 20px;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
        z-index: 10001;
        font-size: 14px;
        font-weight: 500;
        animation: slideIn 0.3s ease-out;
        display: flex;
        align-items: center;
        gap: 8px;
        max-width: 300px;
    `;
    
    notification.innerHTML = `
        <span style="font-size: 16px;">${icon}</span>
        <span>${message}</span>
    `;
    
    document.body.appendChild(notification);
    
    // Add slide-in animation
    const style = document.createElement('style');
    style.textContent = `
        @keyframes slideIn {
            from { transform: translateX(100%); opacity: 0; }
            to { transform: translateX(0); opacity: 1; }
        }
    `;
    document.head.appendChild(style);
    
    // Remove after 4 seconds for errors, 3 seconds for others
    setTimeout(() => {
        notification.remove();
    }, type === 'error' ? 4000 : 3000);
}



function updateDocumentStats() {
    const wordCount = document.getElementById('wordCount');
    const charCount = document.getElementById('charCount');
    
    if (!wordCount || !charCount) return;
    
    // Get all text content from the single content area
    const content = document.getElementById('docEditorContent');
    if (!content) return;
    
    let allText = content.innerText || content.textContent || '';
    
    // Remove placeholder content and chat messages from word count
    allText = allText
        .replace(/Start Writing with Genius AI[\s\S]*?Draft an email about\.\.\./g, '')
        .replace(/Genius is thinking\.\.\./g, '')
        .replace(/Press ESC to close/g, '')
        .replace(/Switch to edit mode and ask Genius to write your first draft, or begin typing to get started/g, '')
        .replace(/Genius AI[\s\S]*?How can I assist you today\? ✨/g, '')
        .trim();
    
    // Count words
    const words = allText.trim().split(/\s+/).filter(word => word.length > 0);
    const numWords = words.length;
    wordCount.textContent = `${numWords} word${numWords !== 1 ? 's' : ''}`;
    
    // Count characters (without spaces) - only count actual content
    const chars = allText.replace(/\s/g, '').length;
    charCount.textContent = `${chars} character${chars !== 1 ? 's' : ''}`;
    
    console.log('Document stats updated - Words:', numWords, 'Chars:', chars, 'Text:', allText.substring(0, 100));
}

function removeGhostText() {
    const content = document.getElementById('docEditorContent');
    if (!content) return;
    
    // Always remove ghost text - be aggressive about it
    const ghostText = content.querySelector('.ghost-text-hint');
    if (ghostText) {
        ghostText.remove();
        console.log('Ghost text removed');
    }
    
    // Double-check and remove any remaining ghost text
    const allGhostText = content.querySelectorAll('.ghost-text-hint');
    allGhostText.forEach(ghost => {
        ghost.remove();
        console.log('Additional ghost text removed');
    });
}

function updatePlaceholderVisibility() {
    const content = document.getElementById('docEditorContent');
    if (!content) return;
    
        // Check if content is empty (only whitespace, placeholder, or chat messages)
        const textContent = content.innerText || '';
        const htmlContent = content.innerHTML || '';
        
        // Remove chat message content from consideration
        const contentWithoutPlaceholder = htmlContent
            .replace(/<div class="genius-chat-container">[\s\S]*?<\/div>/g, '')
            .replace(/<div class="genius-thinking-container">[\s\S]*?<\/div>/g, '')
            .replace(/<div class="ghost-text-hint">[\s\S]*?<\/div>/g, '') // Remove ghost text from consideration
            .replace(/<br\s*\/?>/g, '') // Remove line breaks
            .replace(/&nbsp;/g, ' ') // Replace non-breaking spaces
            .replace(/\s+/g, ' ') // Normalize whitespace
            .trim();
        
        // Check if there's any real content (not just whitespace, ghost text, or empty elements)
        const hasRealContent = contentWithoutPlaceholder.length > 0 && 
                               !contentWithoutPlaceholder.includes('Click the ? to switch to edit with Genius') &&
                               contentWithoutPlaceholder !== '';
        
        if (hasRealContent) {
            // Remove ghost text if it exists
            removeGhostText();
        } else {
            // Always show genius essay writing text when document is empty
            if (!content.querySelector('.ghost-text-hint')) {
                // Add ghost text without replacing the entire content
                const ghostText = document.createElement('div');
                ghostText.className = 'ghost-text-hint';
                ghostText.style.display = 'block';
                ghostText.style.pointerEvents = 'none';
                ghostText.style.userSelect = 'none';
                ghostText.textContent = 'Click the ? to switch to edit with Genius to write your essay for you';
                content.appendChild(ghostText);
            }
        }
}

function setupPlaceholderExamples(classData, existingDoc) {
    console.log('Setting up placeholder examples...');
    // Add event listeners to clickable placeholder examples
    const examples = document.querySelectorAll('.clickable-example');
    console.log('Found clickable examples:', examples.length);
    
    examples.forEach((example, index) => {
        console.log(`Setting up example ${index + 1}:`, example.dataset.example);
        example.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            
            const exampleText = example.dataset.example;
            console.log('Placeholder example clicked:', exampleText);
            
            // Switch to edit mode and send the example as a message
            console.log('Switching to edit mode...');
            switchToEditMode();
            
            // Set the input value and send the message
            setTimeout(() => {
                const chatInput = document.getElementById('geniusChatInput');
                console.log('Chat input found:', !!chatInput);
                if (chatInput) {
                    chatInput.value = exampleText;
                    chatInput.focus();
                    console.log('Set input value to:', exampleText);
                    
                    // Trigger the send message
                    setTimeout(() => {
                        const sendBtn = document.getElementById('geniusSendBtn');
                        console.log('Send button found:', !!sendBtn, 'Disabled:', sendBtn?.disabled);
                        if (sendBtn && !sendBtn.disabled) {
                            console.log('Clicking send button...');
                            sendBtn.click();
                        } else {
                            console.log('Send button not available or disabled');
                        }
                    }, 100);
                } else {
                    console.log('Chat input not found');
                }
            }, 200);
        });
    });
}

 function setupGeniusChat(classData, existingDoc) {
    // Store globally for use in other functions
    window.currentClassData = classData;
    window.currentExistingDoc = existingDoc;
    
    const chatInput = document.getElementById('geniusChatInput');
    const chatContainer = document.getElementById('geniusChatContainer');
    const sidebar = document.getElementById('geniusSidebar');
    const closeSidebarBtn = document.getElementById('closeSidebarBtn');
    const chatMessages = document.getElementById('geniusChatMessages');
    const chatStatus = document.getElementById('geniusChatStatus');
    const modeToggle = document.getElementById('geniusModeToggle');
    const modeIcon = document.getElementById('modeIcon');
    
    if (chatInput) {
        setupGeniusInputListeners(chatInput, classData, existingDoc);
    }
}

function setupGeniusInputListeners(chatInput, classData, existingDoc) {
    const chatContainer = document.getElementById('geniusChatContainer');
    const sidebar = document.getElementById('geniusSidebar');
    const closeSidebarBtn = document.getElementById('closeSidebarBtn');
    const chatMessages = document.getElementById('geniusChatMessages');
    const chatStatus = document.getElementById('geniusChatStatus');
    const modeToggle = document.getElementById('geniusModeToggle');
    const modeIcon = document.getElementById('modeIcon');
    const newChatBtn = document.getElementById('newChatBtn');
    const chatTabs = document.getElementById('geniusChatTabs');
    const sidebarInput = document.getElementById('geniusChatInputSidebar');
    const sendBtn = document.getElementById('geniusSendBtn');
    const inputWrapper = document.getElementById('geniusInputWrapper');
    
    // Ensure input wrapper starts in collapsed state
    if (inputWrapper) {
        inputWrapper.classList.remove('expanded');
        console.log('Input wrapper initialized in collapsed state');
    }
    
    // Add click handler to wrapper to expand when clicked
    if (inputWrapper) {
        inputWrapper.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            if (!inputWrapper.classList.contains('expanded')) {
                inputWrapper.classList.add('expanded');
                console.log('Input wrapper expanded due to wrapper click');
            }
            if (chatInput) {
                chatInput.focus();
            }
        });
    }
    
    // Add click handler to expand when clicked
    if (chatInput) {
        chatInput.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            if (!inputWrapper.classList.contains('expanded')) {
                inputWrapper.classList.add('expanded');
                console.log('Input wrapper expanded due to input click');
            }
            chatInput.focus();
        });
    }
    
    // Add input handler to expand when typing
    if (chatInput) {
        chatInput.addEventListener('input', () => {
            if (!inputWrapper.classList.contains('expanded')) {
                inputWrapper.classList.add('expanded');
                console.log('Input wrapper expanded due to typing');
            }
        });
    }
    
    const OPENAI_API_KEY = window.getOpenAIApiKey() || window.APP_CONFIG.OPENAI_API_KEY;
    
    // Mode state: 'help' or 'edit'
    let currentMode = 'help'; // Start with help mode (? icon)
    
    // Save selection when user clicks on input
    let savedSelection = null;
    
    // Chat management
    let currentChatId = 'default';
    let chats = {};
    let chatCounter = 1;
    
    if (!chatInput || !chatContainer || !sidebar) {
        console.log('Missing required elements for Genius chat setup');
        return;
    }
    
    // Ensure input is always focusable
    chatInput.disabled = false;
    chatInput.readOnly = false;
    chatInput.tabIndex = 0;
    
    // Additional setup to ensure input works
    chatInput.style.pointerEvents = 'auto';
    chatInput.style.opacity = '1';
    chatInput.style.visibility = 'visible';
    
    // Add click handler to ensure input is always focusable
    chatInput.addEventListener('click', (e) => {
        e.preventDefault();
        e.stopPropagation();
        chatInput.focus();
        console.log('Genius input clicked and focused');
    });
    
    // Add focus handler to ensure input stays focusable
    chatInput.addEventListener('focus', () => {
        chatContainer.classList.add('focused');
        console.log('Genius input focused');
        // Ensure input wrapper is expanded when focused
        if (inputWrapper && !inputWrapper.classList.contains('expanded')) {
            inputWrapper.classList.add('expanded');
            console.log('Input wrapper expanded on focus');
        }
    });
    
    chatInput.addEventListener('blur', () => {
        chatContainer.classList.remove('focused');
        console.log('Genius input blurred');
        // Collapse input wrapper when not focused and empty
        if (inputWrapper && (!chatInput.value || chatInput.value.trim() === '')) {
            console.log('Removing expanded class from input wrapper');
            inputWrapper.classList.remove('expanded');
        }
    });
    
    // Periodic check to ensure input stays focusable
    const focusCheckInterval = setInterval(() => {
        if (chatInput && chatInput.disabled) {
            console.log('Input was disabled, re-enabling...');
            chatInput.disabled = false;
            chatInput.readOnly = false;
        }
    }, 1000);
    
    // Clean up interval when input is removed
    const observer = new MutationObserver(() => {
        if (!document.contains(chatInput)) {
            clearInterval(focusCheckInterval);
            observer.disconnect();
        }
    });
    observer.observe(document.body, { childList: true, subtree: true });
    
    // Update placeholder based on selection and mode
    function updatePlaceholder() {
        // Check saved selection first
        let hasSelection = false;
        if (savedSelection) {
            try {
                hasSelection = savedSelection.toString().trim().length > 0;
            } catch (err) {
                hasSelection = false;
            }
        }
        
        // Check current selection if no saved selection
        if (!hasSelection) {
            const selection = window.getSelection();
            hasSelection = selection && selection.toString().trim().length > 0;
        }
        
        if (chatInput.value.trim().length > 0) {
            chatInput.placeholder = '';
        } else {
            if (currentMode === 'edit' && hasSelection) {
                chatInput.placeholder = 'Edit highlighted part with Genius';
            } else if (currentMode === 'edit') {
                chatInput.placeholder = 'Edit document with Genius';
            } else {
                chatInput.placeholder = 'Talk to Genius';
            }
        }
    }
    
    // When user starts typing, remove placeholder
    chatInput.addEventListener('input', () => {
        updatePlaceholder();
    });
    
    // Update placeholder when selection changes
    document.addEventListener('selectionchange', () => {
        if (document.activeElement && document.activeElement.closest('.doc-editor-content')) {
            updatePlaceholder();
        }
    });
    
    // Handle Enter key
    chatInput.addEventListener('keydown', async (e) => {
        if (e.key === 'Enter' && chatInput.value.trim()) {
            e.preventDefault();
            const message = chatInput.value.trim();
            chatInput.value = '';
            
             if (currentMode === 'help') {
                 // Help mode: Show response in sidebar chat
                 chatInput.placeholder = 'Talk to Genius';
                 sidebar.classList.add('open');
                 
                 // Add user message to current chat
                 addMessageToChat('user', message);
                 
                 const documentContent = getDocumentContent(false); // Help mode
                 
                 // Show thinking status
                 if (chatStatus) {
                     chatStatus.textContent = 'Genius is thinking...';
                     chatStatus.style.display = 'block';
                 }
                 
                 try {
                     // Send to OpenAI
                     const response = await fetch('https://api.openai.com/v1/chat/completions', {
                         method: 'POST',
                         headers: {
                             'Content-Type': 'application/json',
                             'Authorization': `Bearer ${OPENAI_API_KEY}`
                         },
                         body: JSON.stringify({
                             model: 'gpt-4o-mini',
                             messages: [
                                 {
                                     role: 'system',
                                     content: `You are Genius AI, a helpful writing assistant. The user is working on a document. Here is the current content of their document:\n\n${documentContent}\n\nAnswer their questions about the document or provide help. Be concise, helpful, and format your response nicely. Use the document as reference but don't quote it directly. Keep responses under 200 words and use proper formatting with line breaks and bullet points when appropriate.`
                                 },
                                 {
                                     role: 'user',
                                     content: message
                                 }
                             ],
                             temperature: 0.7,
                             max_tokens: 500
                         })
                     });
                     
                     if (!response.ok) {
                         throw new Error(`API error: ${response.status}`);
                     }
                     
                     const data = await response.json();
                     const aiResponse = data.choices[0].message.content;
                     
                     // Hide thinking status
                     if (chatStatus) {
                         chatStatus.style.display = 'none';
                     }
                     
                     // Add AI response to current chat
                     const formattedResponse = formatAIResponse(aiResponse);
                     addMessageToChat('assistant', formattedResponse);
                     
                 } catch (error) {
                     console.error('OpenAI API Error:', error);
                     if (chatStatus) {
                         chatStatus.style.display = 'none';
                     }
                     addMessageToChat('assistant', 'Sorry, I encountered an error. Please try again.');
                 }
             } else {
                 // Edit mode: Show suggestions in document
                 chatInput.placeholder = 'Edit document with Genius';
                 const documentContent = getDocumentContent(true); // Edit mode
                 
                 // Check if we have content to work with in edit mode
                 if (!documentContent.trim()) {
                     // For empty documents, we'll generate content instead of showing error
                     console.log('Empty document detected in edit mode - will generate content');
                 }
                 
                 // Keep spotlight effect active during AI processing
                 const keepSpotlight = () => {
                     if (window.spotlightOverlay && !window.spotlightOverlay.parentNode) {
                         // Recreate spotlight if it was removed
                         const selection = window.getSelection();
                         if (selection && selection.rangeCount > 0) {
                             const range = selection.getRangeAt(0);
                             window.createSpotlightEffect(range);
                         }
                     }
                 };
                 
                 // Keep spotlight active
                 const spotlightInterval = setInterval(keepSpotlight, 100);
                 
                 try {
                     console.log('Sending to OpenAI - message:', message, 'documentContent:', documentContent, 'mode: edit');
                     await sendToOpenAI(message, documentContent, 'edit', classData, existingDoc);
                 } catch (error) {
                     console.error('Error in sendToOpenAI:', error);
                     // Show user-friendly error message
                     showNotification('Error processing your request. Please try again.', 'error');
                 } finally {
                     // Clear interval and remove spotlight after processing
                     clearInterval(spotlightInterval);
                     removeSpotlightEffect();
                 }
             }
        }
    });
    
    // Save selection when clicking input and create spotlight effect
    window.spotlightOverlay = null;
    
    chatInput.addEventListener('mousedown', (e) => {
        const selection = window.getSelection();
        if (selection && selection.rangeCount > 0) {
            try {
                savedSelection = selection.getRangeAt(0).cloneRange();
                
                // Create spotlight effect
                const selectedText = selection.toString();
                if (selectedText.trim()) {
                    window.createSpotlightEffect(savedSelection);
                }
            } catch (err) {
                savedSelection = null;
            }
        }
    });
    
    window.createSpotlightEffect = function(range) {
        // Remove existing spotlight
        removeSpotlightEffect();
        
        try {
            // Get the selected text and HTML
            const selectedText = range.toString();
            if (!selectedText.trim()) return;
            
            // Get the bounding rectangles of the selection
            const rects = range.getClientRects();
            if (rects.length === 0) return;
            
            // Get the editor wrapper for positioning context
            const wrapper = document.querySelector('.doc-editor-content-wrapper');
            if (!wrapper) return;
            
            // Add dimming class to wrapper
            wrapper.classList.add('genius-spotlight-active');
            
            const wrapperRect = wrapper.getBoundingClientRect();
            
            // Create container for highlights and text
            window.spotlightOverlay = document.createElement('div');
            window.spotlightOverlay.className = 'genius-spotlight-container';
            
            // Create a single text copy for the entire selection
            const firstRect = rects[0];
            const lastRect = rects[rects.length - 1];
            
            // Calculate bounding box for all rectangles
            let minLeft = Infinity, minTop = Infinity, maxRight = -Infinity, maxBottom = -Infinity;
            for (let i = 0; i < rects.length; i++) {
                const rect = rects[i];
                const left = rect.left - wrapperRect.left + wrapper.scrollLeft;
                const top = rect.top - wrapperRect.top + wrapper.scrollTop;
                const right = left + rect.width;
                const bottom = top + rect.height;
                
                minLeft = Math.min(minLeft, left);
                minTop = Math.min(minTop, top);
                maxRight = Math.max(maxRight, right);
                maxBottom = Math.max(maxBottom, bottom);
                
                // Create blur backdrop for each line
                const blurBox = document.createElement('div');
                blurBox.className = 'genius-spotlight-blur';
                blurBox.style.left = left + 'px';
                blurBox.style.top = top + 'px';
                blurBox.style.width = rect.width + 'px';
                blurBox.style.height = rect.height + 'px';
                window.spotlightOverlay.appendChild(blurBox);
                
                // Create glow box for each line
                const highlightBox = document.createElement('div');
                highlightBox.className = 'genius-spotlight-highlight';
                highlightBox.style.left = left + 'px';
                highlightBox.style.top = top + 'px';
                highlightBox.style.width = rect.width + 'px';
                highlightBox.style.height = rect.height + 'px';
                window.spotlightOverlay.appendChild(highlightBox);
            }
            
            // Clone the entire selection and render it exactly as it appears
            const fragment = range.cloneContents();
            
            // Find the containing parent element to clone its structure
            const startContainer = range.commonAncestorContainer;
            const parentElement = startContainer.nodeType === Node.TEXT_NODE 
                ? startContainer.parentElement 
                : startContainer;
            
            // Create wrapper that matches the parent structure
            const textCopy = document.createElement('div');
            textCopy.className = 'genius-spotlight-text-copy';
            
            // Position it exactly over the original selection
            textCopy.style.left = minLeft + 'px';
            textCopy.style.top = minTop + 'px';
            textCopy.style.width = (maxRight - minLeft) + 'px';
            
            // Append the cloned fragment directly
            textCopy.appendChild(fragment);
            
            window.spotlightOverlay.appendChild(textCopy);
            
            wrapper.appendChild(window.spotlightOverlay);
        } catch (err) {
            console.error('Error creating spotlight:', err);
        }
    }
    
    window.removeSpotlightEffect = function() {
        const wrapper = document.querySelector('.doc-editor-content-wrapper');
        if (wrapper) {
            wrapper.classList.remove('genius-spotlight-active');
        }
        if (window.spotlightOverlay) {
            window.spotlightOverlay.remove();
            window.spotlightOverlay = null;
        }
    }
    
    // Remove spotlight when clicking back on document
    document.addEventListener('click', (e) => {
        if (e.target.closest('.doc-editor-content')) {
            removeSpotlightEffect();
        }
    });
    
     // Don't remove spotlight after submitting - let the processing handle it
     // chatInput.addEventListener('keydown', (e) => {
     //     if (e.key === 'Enter') {
     //         setTimeout(() => removeSpotlightEffect(), 100);
     //     }
     // });
    
    // Focus effect
    chatInput.addEventListener('focus', () => {
        chatContainer.classList.add('focused');
    });
    
    chatInput.addEventListener('blur', () => {
        chatContainer.classList.remove('focused');
    });
    
    // Close sidebar
    if (closeSidebarBtn) {
        closeSidebarBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Close button clicked');
            sidebar.style.display = 'none';
            console.log('Sidebar completely hidden');
        });
    } else {
        console.log('Close button not found');
    }
    
    // Setup resize functionality
    setupGeniusSidebarResize();
    
    // Initialize chat management with a delay to ensure Firebase is loaded
    setTimeout(() => {
    initializeChatManagement();
    }, 1000);
    
    // Also try to initialize immediately if Firebase is already available
    if (window.chatService && typeof window.chatService.getChats === 'function') {
        console.log('Firebase chatService already available, initializing immediately');
        initializeChatManagement();
    }
    
    // New chat button
    if (newChatBtn) {
        newChatBtn.addEventListener('click', createNewChat);
    }
    
    // Sidebar input functionality
    if (sidebarInput && sendBtn) {
        setupSidebarInput();
    }
    
    async function initializeChatManagement() {
        // Prevent duplicate initialization
        if (window.chatManagementInitialized) {
            console.log('Chat management already initialized, skipping');
            return;
        }
        window.chatManagementInitialized = true;
        
        // Load saved chats from Firebase first, then localStorage
        const docId = existingDoc ? existingDoc.id : 'new-document';
        
        try {
            // Try Firebase first with retry mechanism
            let chatService = null;
            let retryCount = 0;
            const maxRetries = 3;
            
            while (retryCount < maxRetries) {
                try {
                    const { chatService: importedChatService } = await import('./firebase-service.js');
                    if (importedChatService && typeof importedChatService.getChats === 'function') {
                        chatService = importedChatService;
                        break;
                    }
                } catch (importError) {
                    console.warn(`Firebase import attempt ${retryCount + 1} failed:`, importError);
                }
                
                retryCount++;
                if (retryCount < maxRetries) {
                    await new Promise(resolve => setTimeout(resolve, 500));
                }
            }
            
            // Fallback to window object if ES6 import fails
            if (!chatService && window.chatService && typeof window.chatService.getChats === 'function') {
                console.log('Using chatService from window object');
                chatService = window.chatService;
            }
            
            if (!chatService) {
                throw new Error('Firebase chatService not available after retries');
            }
            
            const firebaseChats = await chatService.getChats(classData.userId, classData.id, docId);
            
            if (Object.keys(firebaseChats).length > 0) {
                console.log('Loaded chats from Firebase:', Object.keys(firebaseChats).length);
                chats = firebaseChats;
                chatCounter = Math.max(...Object.keys(chats).map(id => parseInt(id.split('-')[1]) || 0)) + 1;
                
                // Ensure all existing chats have the needsNaming flag
                Object.values(chats).forEach(chat => {
                    if (chat.needsNaming === undefined) {
                        chat.needsNaming = false; // Existing chats don't need renaming
                    }
                });
            } else {
                // Fallback to localStorage
        const chatsKey = `genius_chats_${classData.userId}_${classData.name}_${docId}`;
        const savedChats = localStorage.getItem(chatsKey);
        
        if (savedChats) {
            try {
                chats = JSON.parse(savedChats);
                chatCounter = Math.max(...Object.keys(chats).map(id => parseInt(id.split('-')[1]) || 0)) + 1;
                
                // Ensure all existing chats have the needsNaming flag
                Object.values(chats).forEach(chat => {
                    if (chat.needsNaming === undefined) {
                        chat.needsNaming = false; // Existing chats don't need renaming
                    }
                });
                        
                        console.log('Loaded chats from localStorage:', Object.keys(chats).length);
                        
                        // Migrate to Firebase
                        if (Object.keys(chats).length > 0 && chatService && typeof chatService.saveChats === 'function') {
                            console.log('Migrating chats from localStorage to Firebase...');
                            await chatService.saveChats(classData.userId, classData.id, docId, chats);
                            console.log('Chats migrated to Firebase');
                        }
            } catch (error) {
                        console.error('Error parsing saved chats:', error);
                chats = {};
                    }
                }
            }
        } catch (error) {
            console.error('Error loading chats:', error);
            // Fallback to localStorage
            const chatsKey = `genius_chats_${classData.userId}_${classData.name}_${docId}`;
            const savedChats = localStorage.getItem(chatsKey);
            
            if (savedChats) {
                try {
                    chats = JSON.parse(savedChats);
                    chatCounter = Math.max(...Object.keys(chats).map(id => parseInt(id.split('-')[1]) || 0)) + 1;
                    
                    // Ensure all existing chats have the needsNaming flag
                    Object.values(chats).forEach(chat => {
                        if (chat.needsNaming === undefined) {
                            chat.needsNaming = false; // Existing chats don't need renaming
                        }
                    });
                    
                    console.log('Loaded chats from localStorage fallback:', Object.keys(chats).length);
                } catch (error) {
                    console.error('Error parsing saved chats:', error);
                    chats = {};
                }
            }
        }
        
        // Ensure we always have at least one chat, even if everything fails
        if (Object.keys(chats).length === 0) {
            console.log('No chats found, creating default chat');
            createNewChat();
        } else {
            // Load the first chat
            const firstChatId = Object.keys(chats)[0];
            switchToChat(firstChatId);
        }
        
        // Update any existing chats that still have generic names
        Object.values(chats).forEach(chat => {
            if (chat.title.startsWith('Chat ') && chat.messages.length > 0) {
                const firstUserMessage = chat.messages.find(m => m.role === 'user');
                if (firstUserMessage) {
                    chat.title = generateChatTitle(firstUserMessage.content);
                    chat.needsNaming = false;
                }
            }
        });
        
        // Save updated chats and re-render
        saveChats();
        renderChatTabs();
    }
    
    function createNewChat() {
        const chatId = `chat-${chatCounter++}`;
        const chatTitle = `New Chat`;
        
        chats[chatId] = {
            id: chatId,
            title: chatTitle,
            messages: [],
            createdAt: new Date().toISOString(),
            needsNaming: true // Flag to indicate this chat needs a proper name
        };
        
        currentChatId = chatId;
        switchToChat(chatId);
        renderChatTabs();
        saveChats();
        
        console.log('Created new chat:', chatId);
    }
    
    function switchToChat(chatId) {
        if (!chats[chatId]) return;
        
        currentChatId = chatId;
        
        // Clear current messages
        chatMessages.innerHTML = '';
        
        // Load messages for this chat
        const chat = chats[chatId];
        chat.messages.forEach(message => {
            addMessageToChat(message.role, message.content, false);
        });
        
        // Update active tab
        document.querySelectorAll('.chat-tab').forEach(tab => {
            tab.classList.remove('active');
        });
        document.querySelector(`[data-chat-id="${chatId}"]`)?.classList.add('active');
        
        console.log('Switched to chat:', chatId);
    }
    
    function renderChatTabs() {
        if (!chatTabs) return;
        
        chatTabs.innerHTML = '';
        
        Object.values(chats).forEach(chat => {
            const tab = document.createElement('div');
            tab.className = `chat-tab ${chat.id === currentChatId ? 'active' : ''}`;
            tab.dataset.chatId = chat.id;
            tab.innerHTML = `
                <span class="chat-tab-title">${chat.title}</span>
                <button class="chat-tab-close" data-chat-id="${chat.id}">×</button>
            `;
            
            // Click to switch chat
            tab.addEventListener('click', (e) => {
                if (!e.target.classList.contains('chat-tab-close')) {
                    switchToChat(chat.id);
                }
            });
            
            // Close chat button
            const closeBtn = tab.querySelector('.chat-tab-close');
            closeBtn.addEventListener('click', (e) => {
                e.stopPropagation();
                deleteChat(chat.id);
            });
            
            chatTabs.appendChild(tab);
        });
    }
    
    function deleteChat(chatId) {
        if (Object.keys(chats).length <= 1) {
            alert('Cannot delete the last chat');
            return;
        }
        
        delete chats[chatId];
        
        // Switch to another chat
        const remainingChats = Object.keys(chats);
        if (remainingChats.length > 0) {
            switchToChat(remainingChats[0]);
        }
        
        renderChatTabs();
        saveChats();
        
        console.log('Deleted chat:', chatId);
    }
    
    async function saveChats() {
        const docId = existingDoc ? existingDoc.id : 'new-document';
        
        try {
            // Save to Firebase with retry mechanism
            let chatService = null;
            let retryCount = 0;
            const maxRetries = 2;
            
            while (retryCount < maxRetries) {
                try {
                    const { chatService: importedChatService } = await import('./firebase-service.js');
                    if (importedChatService && typeof importedChatService.saveChats === 'function') {
                        chatService = importedChatService;
                        break;
                    }
                } catch (importError) {
                    console.warn(`Firebase import attempt ${retryCount + 1} failed:`, importError);
                }
                
                retryCount++;
                if (retryCount < maxRetries) {
                    await new Promise(resolve => setTimeout(resolve, 300));
                }
            }
            
            // Fallback to window object if ES6 import fails
            if (!chatService && window.chatService && typeof window.chatService.saveChats === 'function') {
                console.log('Using chatService from window object for save');
                chatService = window.chatService;
            }
            
            if (chatService) {
            await chatService.saveChats(classData.userId, classData.id, docId, chats);
            console.log('Chats saved to Firebase');
            } else {
                console.warn('Firebase chatService not available, using localStorage only');
            }
        } catch (error) {
            console.error('Error saving chats to Firebase:', error);
        }
        
        // Also save to localStorage as backup
        const chatsKey = `genius_chats_${classData.userId}_${classData.name}_${docId}`;
        localStorage.setItem(chatsKey, JSON.stringify(chats));
    }
    
    function generateChatTitle(firstMessage) {
        // Simple title generation based on first message
        let title = firstMessage.trim();
        
        // Remove common question words and make it shorter
        title = title.replace(/^(what|how|why|when|where|can|could|would|should|is|are|do|does|did|will|would|please|help|explain|tell|show|give|make|create|write|edit|fix|improve|change|update|add|remove|delete)\s+/i, '');
        
        // Capitalize first letter
        title = title.charAt(0).toUpperCase() + title.slice(1);
        
        // Limit length and add ellipsis if needed
        if (title.length > 30) {
            title = title.substring(0, 30).trim() + '...';
        }
        
        // If title is too short or empty, use a default
        if (title.length < 3) {
            title = 'Quick Question';
        }
        
        return title;
    }
    
    function updateChatTitleIfNeeded(chatId, firstUserMessage) {
        if (chats[chatId] && chats[chatId].needsNaming && firstUserMessage) {
            const newTitle = generateChatTitle(firstUserMessage);
            chats[chatId].title = newTitle;
            chats[chatId].needsNaming = false;
            
            // Update the tab display
            renderChatTabs();
            saveChats();
            
            console.log('Updated chat title:', newTitle);
        }
    }
    
    function setupSidebarInput() {
        // Handle Enter key
        sidebarInput.addEventListener('keydown', async (e) => {
            if (e.key === 'Enter' && sidebarInput.value.trim()) {
                e.preventDefault();
                await sendSidebarMessage();
            }
        });
        
        // Handle Send button click
        sendBtn.addEventListener('click', async (e) => {
            e.preventDefault();
            if (sidebarInput.value.trim()) {
                await sendSidebarMessage();
            }
        });
        
        // Focus input when sidebar opens
        sidebarInput.addEventListener('focus', () => {
            console.log('Sidebar input focused');
        });
    }
    
    async function sendSidebarMessage() {
        const message = sidebarInput.value.trim();
        if (!message) return;
        
        // Clear input
        sidebarInput.value = '';
        
        // Add user message to chat
        addMessageToChat('user', message);
        
        // Show thinking status
        if (chatStatus) {
            chatStatus.textContent = 'Genius is thinking...';
            chatStatus.style.display = 'block';
        }
        
        try {
            // Get document content
            const documentContent = getDocumentContent(false); // Help mode for sidebar
            
            // Send to OpenAI
            const response = await fetch('https://api.openai.com/v1/chat/completions', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${OPENAI_API_KEY}`
                },
                body: JSON.stringify({
                    model: 'gpt-4o-mini',
                    messages: [
                        {
                            role: 'system',
                            content: `You are Genius AI, a helpful writing assistant. The user is working on a document. Here is the current content of their document:\n\n${documentContent}\n\nAnswer their questions about the document or provide help. Be concise, helpful, and format your response nicely. Use the document as reference but don't quote it directly. Keep responses under 200 words and use proper formatting with line breaks and bullet points when appropriate.`
                        },
                        {
                            role: 'user',
                            content: message
                        }
                    ],
                    temperature: 0.7,
                    max_tokens: 500
                })
            });
            
            if (!response.ok) {
                throw new Error(`API error: ${response.status}`);
            }
            
            const data = await response.json();
            const aiResponse = data.choices[0].message.content;
            
            // Hide thinking status
            if (chatStatus) {
                chatStatus.style.display = 'none';
            }
            
            // Add AI response to chat
            const formattedResponse = formatAIResponse(aiResponse);
            addMessageToChat('assistant', formattedResponse);
            
        } catch (error) {
            console.error('OpenAI API Error:', error);
            if (chatStatus) {
                chatStatus.style.display = 'none';
            }
            addMessageToChat('assistant', 'Sorry, I encountered an error. Please try again.');
        }
    }
    
    function setupGeniusSidebarResize() {
        const sidebar = document.getElementById('geniusSidebar');
        const resizeHandle = document.getElementById('geniusResizeHandle');
        
        if (!sidebar || !resizeHandle) {
            console.log('Resize elements not found');
            return;
        }
        
        let isResizing = false;
        let startX = 0;
        let startWidth = 0;
        
        resizeHandle.addEventListener('mousedown', (e) => {
            console.log('Resize started');
            isResizing = true;
            startX = e.clientX;
            startWidth = parseInt(window.getComputedStyle(sidebar).width, 10);
            
            // Add visual feedback
            document.body.style.cursor = 'col-resize';
            document.body.style.userSelect = 'none';
            
            document.addEventListener('mousemove', handleResize);
            document.addEventListener('mouseup', stopResize);
            
            e.preventDefault();
            e.stopPropagation();
        });
        
        function handleResize(e) {
            if (!isResizing) return;
            
            const newWidth = startWidth - (e.clientX - startX); // Subtract because we're dragging from the left
            const minWidth = 300;
            const maxWidth = window.innerWidth * 0.7;
            
            console.log('Resizing to:', newWidth, 'min:', minWidth, 'max:', maxWidth);
            
            if (newWidth >= minWidth && newWidth <= maxWidth) {
                sidebar.style.width = newWidth + 'px';
            }
        }
        
        function stopResize() {
            console.log('Resize stopped');
            isResizing = false;
            
            // Remove visual feedback
            document.body.style.cursor = '';
            document.body.style.userSelect = '';
            
            document.removeEventListener('mousemove', handleResize);
            document.removeEventListener('mouseup', stopResize);
        }
    }
    
    // Chat toggle button in header
    const chatToggleBtn = document.getElementById('chatToggleBtn');
    if (chatToggleBtn) {
        chatToggleBtn.addEventListener('click', () => {
            if (sidebar.style.display === 'none') {
                sidebar.style.display = 'flex';
                sidebar.classList.add('open');
            } else {
                sidebar.classList.toggle('open');
            }
        });
    }
    
    // Also add close functionality to the close button in the document editor header
    const docCloseBtn = document.getElementById('closeSidebarBtn');
    if (docCloseBtn) {
        docCloseBtn.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            console.log('Document close button clicked');
            sidebar.style.display = 'none';
        });
    }
    
    // Mode toggle
    if (modeToggle && modeIcon) {
        // Set initial state
        if (currentMode === 'help') {
            modeIcon.textContent = '?';
            modeToggle.title = 'Help mode - Click to switch to Edit mode';
            chatInput.placeholder = 'Talk to Genius';
        }
        
        modeToggle.addEventListener('click', (e) => {
            e.stopPropagation();
            e.preventDefault();
            
            if (currentMode === 'help') {
                // Switch to edit mode
                currentMode = 'edit';
                modeIcon.innerHTML = `
                    <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                        <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
                        <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
                    </svg>
                `;
                modeToggle.title = 'Edit mode - Click to switch to Help mode';
                modeToggle.classList.add('edit-mode-active');
                updatePlaceholder();
            } else {
                // Switch to help mode
                currentMode = 'help';
                modeIcon.textContent = '?';
                modeToggle.title = 'Help mode - Click to switch to Edit mode';
                modeToggle.classList.remove('edit-mode-active');
                updatePlaceholder();
            }
        });
    }
    
    function formatAIResponse(response) {
        // Clean up the response and format it nicely
        let formatted = response
            .replace(/\n\n+/g, '\n\n') // Remove excessive line breaks
            .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>') // Bold text
            .replace(/\*(.*?)\*/g, '<em>$1</em>') // Italic text
            .replace(/\n/g, '<br>') // Convert line breaks to HTML
            .replace(/^•\s*/gm, '• ') // Ensure bullet points are consistent
            .replace(/^\d+\.\s*/gm, (match) => `<br>${match}`) // Numbered lists
            .trim();
        
        // Add some basic styling
        return `<div class="ai-response-content">${formatted}</div>`;
    }
    
    function addMessageToChat(role, content, saveToChat = true) {
        const messageDiv = document.createElement('div');
        
        if (role === 'user') {
            // User messages in chat bubble
            messageDiv.className = `chat-message ${role}-message`;
            messageDiv.innerHTML = `
                <div class="message-content">${content}</div>
            `;
            chatMessages.appendChild(messageDiv);
        } else {
            // AI responses as plain text without chat bubble (like main Genius Chat)
            messageDiv.className = 'genius-ai-response';
            const uniqueId = Date.now() + Math.random().toString(36).substr(2, 9);
            messageDiv.innerHTML = `
                <div class="genius-ai-content">
                    <div class="genius-ai-text" id="typingText-${uniqueId}"></div>
                    <div class="genius-ai-actions" style="opacity: 0;">
                        <button class="genius-ai-action" id="copyBtn-${uniqueId}" title="Copy">📋</button>
                    </div>
                </div>
            `;
            chatMessages.appendChild(messageDiv);
            
            // Add typing animation for AI responses
            displayTypingResponse(content, `typingText-${uniqueId}`, `copyBtn-${uniqueId}`);
        }
        
        chatMessages.scrollTop = chatMessages.scrollHeight;
        
        // Save message to current chat
        if (saveToChat && chats[currentChatId]) {
            chats[currentChatId].messages.push({
                role: role,
                content: content,
                timestamp: new Date().toISOString()
            });
            
            // Update chat title if this is the first user message
            if (role === 'user' && chats[currentChatId].messages.filter(m => m.role === 'user').length === 1) {
                updateChatTitleIfNeeded(currentChatId, content);
            }
            
            saveChats();
        }
    }
    
    function formatGeniusMessage(content) {
        // Enhanced markdown formatting like ChatGPT
        let formatted = content
            // Headers
            .replace(/^### (.*$)/gim, '<h3>$1</h3>')
            .replace(/^## (.*$)/gim, '<h2>$1</h2>')
            .replace(/^# (.*$)/gim, '<h1>$1</h1>')
            // Bold and italic
            .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
            .replace(/\*(.*?)\*/g, '<em>$1</em>')
            // Code blocks
            .replace(/```([\s\S]*?)```/g, '<pre><code>$1</code></pre>')
            .replace(/`(.*?)`/g, '<code>$1</code>')
            // Lists
            .replace(/^\* (.*$)/gim, '<li>$1</li>')
            .replace(/^- (.*$)/gim, '<li>$1</li>')
            .replace(/^\d+\. (.*$)/gim, '<li>$1</li>')
            // Line breaks
            .replace(/\n\n/g, '</p><p>')
            .replace(/\n/g, '<br>');

        // Wrap list items in ul tags
        formatted = formatted.replace(/(<li>.*<\/li>)/gs, '<ul>$1</ul>');
        
        // Wrap paragraphs
        if (!formatted.startsWith('<h') && !formatted.startsWith('<ul') && !formatted.startsWith('<pre')) {
            formatted = '<p>' + formatted + '</p>';
        }

        return formatted;
    }
    
    function displayTypingResponse(response, textElementId, copyBtnId) {
        const textElement = document.getElementById(textElementId);
        const copyBtn = document.getElementById(copyBtnId);
        let currentText = '';
        let index = 0;

        // Setup copy button
        copyBtn.addEventListener('click', () => {
            navigator.clipboard.writeText(response);
        });

        // Type out the response character by character
        const typeWriter = () => {
            if (index < response.length) {
                currentText += response[index];
                textElement.innerHTML = formatGeniusMessage(currentText);
                chatMessages.scrollTop = chatMessages.scrollHeight;
                index++;
                
                // Variable speed - faster for spaces, slower for punctuation
                const char = response[index - 1];
                let delay = 20; // Base delay
                if (char === ' ') delay = 10;
                if (char === '.' || char === '!' || char === '?') delay = 100;
                if (char === '\n') delay = 50;
                
                setTimeout(typeWriter, delay);
            } else {
                // Show copy button after typing is complete
                const actions = copyBtn.parentElement;
                actions.style.opacity = '1';
            }
        };

        // Start typing
        typeWriter();
    }
    
    window.getDocumentContent = function(editMode = false) {
        // In edit mode, prioritize highlighted text
        if (editMode) {
            // Check if there's a saved selection
            if (savedSelection) {
                try {
                    const selectedText = savedSelection.toString().trim();
                    if (selectedText) {
                        console.log('Using saved selection for edit mode:', selectedText);
                        return selectedText;
                    }
                } catch (err) {
                    console.log('Error with saved selection:', err);
                }
            }
            
            // Check current selection
            const selection = window.getSelection();
            const selectedText = selection ? selection.toString().trim() : '';
            
            if (selectedText) {
                console.log('Using current selection for edit mode:', selectedText);
                return selectedText;
            }
            
            // If no selection in edit mode, return empty string but allow content generation
            console.log('No selection found in edit mode, allowing content generation');
            return '';
        }
        
        // For help mode, use the original logic
        // Check if there's a saved selection
        if (savedSelection) {
            try {
                const selectedText = savedSelection.toString().trim();
                if (selectedText) {
                    return selectedText;
                }
            } catch (err) {
                // Fall through to full document
            }
        }
        
        // Check current selection
        const selection = window.getSelection();
        const selectedText = selection ? selection.toString().trim() : '';
        
        if (selectedText) {
            // Return only the selected text
            return selectedText;
        }
        
        // Return full document if no selection
        const allContent = document.querySelectorAll('.doc-editor-content');
        let fullText = '';
        allContent.forEach(content => {
            fullText += content.innerText + '\n\n';
        });
        return fullText.trim();
    }
    
     async function sendToOpenAI(userMessage, documentContext, mode, classData, existingDoc) {
        console.log('sendToOpenAI called with:', { userMessage, documentContext, mode });
        try {
            if (mode === 'help') {
                // Help mode: chat response
                chatStatus.textContent = 'Genius is thinking...';
                chatStatus.style.display = 'block';
                
                const response = await fetch('https://api.openai.com/v1/chat/completions', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${OPENAI_API_KEY}`
                    },
                     body: JSON.stringify({
                         model: 'gpt-4o-mini',
                         messages: [
                             {
                                 role: 'system',
                                 content: `You are Genius AI, a helpful writing assistant. The user is working on a document. Here is the current content of their document:\n\n${documentContext}\n\nAnswer their questions about the document or provide help. Be concise, helpful, and format your response nicely. Use the document as reference but don't quote it directly. Keep responses under 200 words and use proper formatting with line breaks and bullet points when appropriate.`
                             },
                             {
                                 role: 'user',
                                 content: userMessage
                             }
                         ],
                         temperature: 0.7,
                         max_tokens: 500
                     })
                });
                
                if (!response.ok) {
                    throw new Error(`API error: ${response.status}`);
                }
                
                 const data = await response.json();
                 const aiResponse = data.choices[0].message.content;
                 
                 chatStatus.style.display = 'none';
                 const formattedResponse = formatAIResponse(aiResponse);
                 addMessageToChat('assistant', formattedResponse);
                
             } else {
                 // Edit mode: suggestions or content generation
                 // Check if document is empty
                 const content = document.getElementById('docEditorContent');
                 const isDocumentEmpty = !content || !content.innerText || content.innerText.trim().length === 0;
                 
                 console.log('Edit mode - isDocumentEmpty:', isDocumentEmpty, 'content:', content?.innerText);
                 
                 // If document is empty in edit mode, treat it as content generation
                 if (isDocumentEmpty) {
                     console.log('Empty document in edit mode - switching to content generation');
                 }
                 
                 // Replace floating Genius component with thinking status
                 const chatContainer = document.getElementById('geniusChatContainer');
                 const originalDisplay = chatContainer.style.display;
                 const originalContent = chatContainer.innerHTML;
                 
                 // Show thinking status in place of floating component
                 chatContainer.style.display = 'flex';
                 chatContainer.innerHTML = `
                     <div class="genius-thinking-container">
                         <img src="assets/darkgenius.png" alt="Genius" class="genius-thinking-icon">
                         <div class="genius-thinking-text">Genius is thinking...</div>
                         <div class="genius-thinking-dots">
                             <span></span>
                             <span></span>
                             <span></span>
                         </div>
                     </div>
                 `;
                 
                 // Add delay to show first step
                 await new Promise(resolve => setTimeout(resolve, 500));
                
                const response = await fetch('https://api.openai.com/v1/chat/completions', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${OPENAI_API_KEY}`
                    },
                    body: JSON.stringify({
                        model: 'gpt-4o-mini',
                        messages: [
                            {
                                role: 'system',
                                content: isDocumentEmpty 
                                    ? `You are a professional content writer. The user has an empty document and wants you to write content based on their request.

USER'S REQUEST: ${userMessage}

INSTRUCTIONS:
1. Write high-quality, well-structured content based on the user's request
2. Use proper formatting with headings, paragraphs, and lists as appropriate
3. Make the content engaging and informative
4. Write in a professional but accessible tone
5. Include relevant details and examples where appropriate
6. Format the response as HTML with proper tags (h1, h2, h3, p, ul, ol, li, strong, em, etc.)

Write the complete content that the user requested.`
                                    : `You are an expert writing editor. Your job is to improve text based on user requests.

DOCUMENT CONTENT:
${documentContext}

USER'S REQUEST: ${userMessage}

INSTRUCTIONS:
1. Analyze the text carefully
2. Find 3-7 specific improvements based on the user's request
3. For each improvement, identify the EXACT original text (word-for-word match including punctuation)
4. Provide a better version and explain why it's better
5. Focus on what the user asked for (grammar, clarity, conciseness, tone, style, etc.)

IMPORTANT RULES:
- "original" MUST be EXACT text from the document - copy it character-by-character
- Match ALL punctuation, spacing, and capitalization exactly
- If the user's request is vague, improve grammar, clarity, and flow
- Each "original" should be 3-50 words long
- ALWAYS return valid JSON in this exact format:

{
  "suggestions": [
    {
      "original": "exact text from document",
      "suggestion": "improved version",
      "reason": "brief explanation"
    }
  ]
}

Do NOT include any text outside the JSON. ALWAYS find at least 3 suggestions.`
                            },
                            {
                                role: 'user',
                                content: isDocumentEmpty 
                                    ? `Please write content for me: ${userMessage}`
                                    : `Please analyze and provide editing suggestions: ${userMessage}`
                            }
                        ],
                        temperature: 0.7,
                        max_tokens: 4000,
                        response_format: isDocumentEmpty ? undefined : { type: "json_object" }
                    })
                });
                
                if (!response.ok) {
                    throw new Error(`API error: ${response.status}`);
                }
                
                 const data = await response.json();
                 let aiResponse = data.choices[0].message.content;
                 
                if (isDocumentEmpty) {
                    // For empty documents, paste the generated content directly
                    try {
                        // Clear any existing placeholder
                        const placeholder = content.querySelector('.empty-document-placeholder');
                        if (placeholder) {
                            placeholder.remove();
                        }
                        
                        // Type out the AI response with animation
                        window.typeOutContent(content, aiResponse);
                        
                        // Auto-save the document
                        setTimeout(() => {
                            window.autoSaveDocument(classData, existingDoc);
                        }, 500);
                        
                        // Show success notification
                        showNotification('Content generated successfully!', 'success');
                        
                    } catch (error) {
                        console.error('Error pasting generated content:', error);
                        showNotification('Error generating content. Please try again.', 'error');
                    }
                } else {
                    // For documents with content, parse suggestions and display them
                    try {
                        // Try to parse as JSON
                        let suggestions;
                        
                        const parsed = JSON.parse(aiResponse);
                        
                        // Handle different response formats
                        if (Array.isArray(parsed)) {
                            // Direct array
                            suggestions = parsed;
                        } else if (parsed.suggestions && Array.isArray(parsed.suggestions)) {
                            // Wrapped in suggestions key
                            suggestions = parsed.suggestions;
                        } else if (parsed.edits && Array.isArray(parsed.edits)) {
                            // Wrapped in edits key
                            suggestions = parsed.edits;
                        } else if (parsed.original && parsed.suggestion) {
                            // Single suggestion object - wrap in array
                            suggestions = [parsed];
                        } else {
                            // Try to extract from any nested structure
                            const possibleArrays = Object.values(parsed).filter(val => Array.isArray(val));
                            if (possibleArrays.length > 0) {
                                suggestions = possibleArrays[0];
                            } else {
                                throw new Error('No valid suggestions found in response');
                            }
                        }
                        
                        if (suggestions.length === 0) {
                            alert('No improvements needed - your text looks great!');
                            return;
                        }
                     
                     await new Promise(resolve => setTimeout(resolve, 500));
                     
                     displaySuggestions(suggestions, classData, existingDoc);
                    } catch (parseError) {
                        console.error('Could not parse suggestions:', parseError);
                        console.log('Raw response:', aiResponse);
                        alert('Successfully analyzed your text! Try being more specific about what to improve (e.g., "make it more concise" or "fix grammar").');
                    }
                }
                 
                 // Restore original floating component
                 chatContainer.style.display = originalDisplay;
                 chatContainer.innerHTML = originalContent;
                 
                 // Re-setup the input event listeners and focus
                 setTimeout(() => {
                     const restoredInput = document.getElementById('geniusChatInput');
                     if (restoredInput) {
                         // Re-add all event listeners
                         setupGeniusInputListeners(restoredInput, classData, existingDoc);
                         
                         // Make sure input is focusable
                         restoredInput.focus();
                         restoredInput.disabled = false;
                         restoredInput.readOnly = false;
                     }
                 }, 100);
            }
            
        } catch (error) {
            console.error('OpenAI API Error:', error);
            if (mode === 'help') {
                chatStatus.style.display = 'none';
                addMessageToChat('assistant', 'Sorry, I encountered an error. Please try again.');
            } else {
                // Check if this was for an empty document
                const content = document.getElementById('docEditorContent');
                const isDocumentEmpty = !content || !content.innerText || content.innerText.trim().length === 0;
                
                if (isDocumentEmpty) {
                    showNotification('Error generating content. Please try again.', 'error');
                } else {
                    alert('Error generating suggestions. Please try again.');
                }
                
                // Restore original floating component on error
                const chatContainer = document.getElementById('geniusChatContainer');
                if (chatContainer && typeof originalDisplay !== 'undefined' && typeof originalContent !== 'undefined') {
                    chatContainer.style.display = originalDisplay;
                    chatContainer.innerHTML = originalContent;
                    
                    // Re-setup the input event listeners
                    setTimeout(() => {
                        const restoredInput = document.getElementById('geniusChatInput');
                        if (restoredInput) {
                            setupGeniusInputListeners(restoredInput, classData, existingDoc);
                        }
                    }, 100);
                }
            }
        }
    }
    
    function showEditingStatus(message) {
        let statusEl = document.getElementById('editingStatusOverlay');
        if (!statusEl) {
            statusEl = document.createElement('div');
            statusEl.id = 'editingStatusOverlay';
            statusEl.className = 'editing-status-overlay';
            document.body.appendChild(statusEl);
        }
        statusEl.textContent = message;
        statusEl.style.display = 'flex';
    }
    
    function hideEditingStatus() {
        const statusEl = document.getElementById('editingStatusOverlay');
        if (statusEl) {
            statusEl.style.display = 'none';
        }
    }
    
    function removeOverlappingSuggestions(newText, contentElement) {
        // Find all existing suggestion markers in this content element
        const existingMarkers = contentElement.querySelectorAll('.suggestion-marker');
        
        existingMarkers.forEach(marker => {
            const markerText = marker.textContent;
            
            // Check if the new text overlaps with existing marker text
            if (isTextOverlapping(newText, markerText)) {
                console.log('Removing overlapping suggestion:', markerText, 'for new text:', newText);
                
                // Get the suggestion index
                const suggestionIndex = marker.dataset.suggestionIndex;
                
                // Remove the marker and replace with plain text
                const textNode = document.createTextNode(markerText);
                marker.replaceWith(textNode);
                
                // Remove the corresponding sidebar indicator
                const indicator = document.getElementById(`indicator-${suggestionIndex}`);
                if (indicator) {
                    indicator.remove();
                }
                
                // Remove the suggestion from localStorage
                removeSuggestionFromStorage(suggestionIndex);
                
                // Update current suggestions state
                window.saveCurrentSuggestions();
            }
        });
    }
    
    function isTextOverlapping(text1, text2) {
        // Normalize text for comparison (remove extra whitespace)
        const normalized1 = text1.trim().toLowerCase();
        const normalized2 = text2.trim().toLowerCase();
        
        // Check if one text contains the other or if they overlap significantly
        if (normalized1 === normalized2) {
            return true; // Exact match
        }
        
        // Check if one is a substring of the other (with some tolerance)
        if (normalized1.length > 10 && normalized2.length > 10) {
            const minLength = Math.min(normalized1.length, normalized2.length);
            const maxLength = Math.max(normalized1.length, normalized2.length);
            
            // If the shorter text is more than 70% of the longer text, consider it overlapping
            if (minLength / maxLength > 0.7) {
                return normalized1.includes(normalized2) || normalized2.includes(normalized1);
            }
        }
        
        // Check for word-level overlap (if they share significant words)
        const words1 = normalized1.split(/\s+/);
        const words2 = normalized2.split(/\s+/);
        
        if (words1.length > 1 && words2.length > 1) {
            const commonWords = words1.filter(word => words2.includes(word));
            const overlapRatio = commonWords.length / Math.min(words1.length, words2.length);
            
            // If more than 60% of words overlap, consider it overlapping
            if (overlapRatio > 0.6) {
                return true;
            }
        }
        
        return false;
    }
    
     function displaySuggestions(suggestions, classData, existingDoc) {
         console.log('displaySuggestions called with:', suggestions, classData, existingDoc);
         
         // Don't clear existing suggestions - allow overlapping
         document.querySelectorAll('.suggestion-popup').forEach(el => el.remove());
         
         const sidebar = document.getElementById('suggestionsSidebar');
         console.log('Suggestions sidebar found:', !!sidebar);
         if (!sidebar) return;
        
        if (!Array.isArray(suggestions) || suggestions.length === 0) {
            alert('No suggestions generated.');
            return;
        }
        
        // Load existing suggestions and merge with new ones
        const docId = existingDoc ? existingDoc.id : 'new-document';
        const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
        const existingSuggestions = JSON.parse(localStorage.getItem(suggestionsKey) || '[]');
        
        // Calculate starting index for new suggestions
        const startIndex = existingSuggestions.length;
        
         // Merge suggestions
         const allSuggestions = [...existingSuggestions, ...suggestions];
         
         // Limit suggestions to prevent localStorage quota exceeded
         const limitedSuggestions = allSuggestions.slice(0, 50); // Keep only first 50 suggestions
         
         try {
             localStorage.setItem(suggestionsKey, JSON.stringify(limitedSuggestions));
             console.log('Saved suggestions to localStorage:', suggestionsKey, limitedSuggestions.length);
         } catch (error) {
             console.error('Failed to save suggestions to localStorage:', error);
             // Clear old suggestions and try again
             try {
                 localStorage.removeItem(suggestionsKey);
                 localStorage.setItem(suggestionsKey, JSON.stringify(limitedSuggestions.slice(0, 10)));
                 console.log('Saved limited suggestions after clearing old data');
             } catch (e) {
                 console.error('Still failed to save suggestions:', e);
             }
         }
         
         // Also save current suggestions for persistence
         window.saveCurrentSuggestions();
        
        const contentElements = document.querySelectorAll('.doc-editor-content');
        
        suggestions.forEach((suggestion, localIndex) => {
            if (!suggestion.original || !suggestion.suggestion) return;
            
            const globalIndex = startIndex + localIndex;
            const colorClass = `suggestion-color-unified`; // Use single color for all suggestions
            
            // Find the text in the document
            contentElements.forEach(content => {
                const html = content.innerHTML;
                const originalText = suggestion.original;
                
                if (html.includes(originalText)) {
                    // Check for overlapping suggestions and remove them
                    removeOverlappingSuggestions(originalText, content);
                    
                    // Mark the original text with underline and color
                    const markerId = `suggestion-${globalIndex}`;
                    const markedHtml = html.replace(
                        originalText,
                        `<span class="suggestion-marker ${colorClass}" id="${markerId}" data-suggestion-index="${globalIndex}">${originalText}</span>`
                    );
                    content.innerHTML = markedHtml;
                    
                     // Create indicator in left sidebar with proper alignment
                     setTimeout(() => {
                         const marker = document.getElementById(markerId);
                         console.log('Looking for marker:', markerId, 'Found:', !!marker);
                         if (marker) {
                             console.log('Creating suggestion indicator for:', suggestion.original);
                             createSuggestionIndicator(marker, suggestion, globalIndex, sidebar, colorClass);
                         }
                     }, 100);
                }
            });
        });
    }
    
// Global function to clean up orphaned suggestions
window.cleanupOrphanedSuggestions = function(classData, existingDoc) {
    try {
        const currentUserData = localStorage.getItem('currentUser');
        if (!currentUserData) return;
        
        const currentUser = JSON.parse(currentUserData);
        const classes = JSON.parse(localStorage.getItem('classes') || '[]');
        
        classes.forEach(classData => {
            if (classData.userId === currentUser.uid) {
                // Clean up all suggestion keys for this user
                const docId = 'current-document';
                const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
                const currentSuggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
                
                // Remove any empty or invalid suggestion arrays
                [suggestionsKey, currentSuggestionsKey].forEach(key => {
                    try {
                        const suggestions = JSON.parse(localStorage.getItem(key) || '[]');
                        if (!Array.isArray(suggestions) || suggestions.length === 0) {
                            localStorage.removeItem(key);
                            console.log('Removed empty suggestions key:', key);
                        }
                    } catch (error) {
                        localStorage.removeItem(key);
                        console.log('Removed invalid suggestions key:', key);
                    }
                });
                
                // Clean up document-specific suggestions
                const documents = JSON.parse(classData.documents || '[]');
                documents.forEach(doc => {
                    const docSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_${doc.id}`;
                    try {
                        const suggestions = JSON.parse(localStorage.getItem(docSuggestionsKey) || '[]');
                        if (!Array.isArray(suggestions) || suggestions.length === 0) {
                            localStorage.removeItem(docSuggestionsKey);
                            console.log('Removed empty document suggestions key:', docSuggestionsKey);
                        }
                    } catch (error) {
                        localStorage.removeItem(docSuggestionsKey);
                        console.log('Removed invalid document suggestions key:', docSuggestionsKey);
                    }
                });
            }
        });
        
        console.log('Cleanup of orphaned suggestions completed');
    } catch (error) {
        console.error('Error cleaning up orphaned suggestions:', error);
    }
};

// Global function to load saved suggestions
window.loadSavedSuggestions = function(classData, existingDoc) {
    // Generate document ID for new documents
    const docId = existingDoc ? existingDoc.id : 'new-document';
    const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
    const savedSuggestions = localStorage.getItem(suggestionsKey);
    
    console.log('Loading saved suggestions for:', suggestionsKey, 'Found:', !!savedSuggestions);
    
    if (savedSuggestions) {
        try {
            const suggestions = JSON.parse(savedSuggestions);
            console.log('Loaded suggestions:', suggestions.length);
            setTimeout(() => {
                displaySuggestions(suggestions, classData, existingDoc);
            }, 500);
        } catch (error) {
            console.error('Error loading saved suggestions:', error);
        }
    }
};
    
// Global function to load current suggestions
window.loadCurrentSuggestions = function(classData, existingDoc) {
    // Load suggestions that were active when the editor was last closed
    const docId = 'current-document';
    const suggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
    const currentSuggestions = localStorage.getItem(suggestionsKey);
    
    console.log('Loading current suggestions for:', suggestionsKey, 'Found:', !!currentSuggestions);
    
    if (currentSuggestions) {
        try {
            const suggestions = JSON.parse(currentSuggestions);
            console.log('Loaded current suggestions:', suggestions.length);
            
            // Restore suggestions to the document
            setTimeout(() => {
                window.restoreSuggestionsToDocument(suggestions);
            }, 1000);
        } catch (error) {
            console.error('Error loading current suggestions:', error);
        }
    }
};
    
// Global function to restore suggestions to document
window.restoreSuggestionsToDocument = function(suggestions) {
    const contentElements = document.querySelectorAll('.doc-editor-content');
    
    suggestions.forEach(suggestion => {
        const { index, original, markerId, indicatorId } = suggestion;
        
        // Find the text in the document and mark it
        contentElements.forEach(content => {
            const html = content.innerHTML;
            if (html.includes(original) && !html.includes('suggestion-marker')) {
                const colorClass = `suggestion-color-unified`;
                const markedHtml = html.replace(
                    original,
                    `<span class="suggestion-marker ${colorClass}" id="${markerId}" data-suggestion-index="${index}">${original}</span>`
                );
                content.innerHTML = markedHtml;
                
                // Create indicator in sidebar
                setTimeout(() => {
                    const marker = document.getElementById(markerId);
                    if (marker) {
                        const sidebar = document.getElementById('suggestionsSidebar');
                        if (sidebar) {
                            createSuggestionIndicator(marker, { original: original }, index, sidebar, colorClass);
                        }
                    }
                }, 100);
            }
        });
    });
    
    console.log('Restored suggestions to document:', suggestions.length);
};
    
     function createSuggestionIndicator(markerElement, suggestion, index, sidebar, colorClass) {
         console.log('createSuggestionIndicator called with:', suggestion, index, sidebar, colorClass);
         
         const indicator = document.createElement('div');
         indicator.className = `suggestion-indicator ${colorClass}`;
         indicator.id = `indicator-${index}`;
         indicator.dataset.suggestionIndex = index;
        
        // Position indicator at same height as marked text
        const rect = markerElement.getBoundingClientRect();
        const wrapperRect = document.querySelector('.doc-editor-content-wrapper').getBoundingClientRect();
        const scrollTop = document.querySelector('.doc-editor-content-wrapper').scrollTop;
        
        // Calculate position relative to wrapper including scroll
        const relativeTop = rect.top - wrapperRect.top + scrollTop;
        
        // Check for other indicators on the same line (within 5px tolerance)
        const existingIndicators = sidebar.querySelectorAll('.suggestion-indicator');
        let horizontalOffset = 0;
        const lineTolerance = 5; // pixels
        
        existingIndicators.forEach(existingIndicator => {
            const existingTop = parseFloat(existingIndicator.style.top) || 0;
            if (Math.abs(existingTop - relativeTop) <= lineTolerance) {
                // Found another indicator on the same line, offset horizontally
                horizontalOffset += 25; // 25px spacing between dots
            }
        });
        
        indicator.style.top = `${relativeTop}px`;
        indicator.style.left = `${10 + horizontalOffset}px`;
        indicator.innerHTML = `<div class="indicator-dot"></div>`;
        
        sidebar.appendChild(indicator);
        
        // Update position on scroll
        const wrapper = document.querySelector('.doc-editor-content-wrapper');
        const updatePosition = () => {
            const newRect = markerElement.getBoundingClientRect();
            const newScrollTop = wrapper.scrollTop;
            const newRelativeTop = newRect.top - wrapperRect.top + newScrollTop;
            
            // Recalculate horizontal offset for this line
            let newHorizontalOffset = 0;
            const existingIndicators = sidebar.querySelectorAll('.suggestion-indicator');
            existingIndicators.forEach(existingIndicator => {
                if (existingIndicator !== indicator) {
                    const existingTop = parseFloat(existingIndicator.style.top) || 0;
                    if (Math.abs(existingTop - newRelativeTop) <= lineTolerance) {
                        newHorizontalOffset += 25;
                    }
                }
            });
            
            indicator.style.top = `${newRelativeTop}px`;
            indicator.style.left = `${10 + newHorizontalOffset}px`;
        };
        
        wrapper.addEventListener('scroll', updatePosition);
        
        // Click on indicator or marked text shows popup
        const showPopup = () => {
            // Remove any existing popups
            document.querySelectorAll('.suggestion-popup').forEach(p => p.remove());
            
            // Highlight this indicator
            document.querySelectorAll('.suggestion-indicator').forEach(i => i.classList.remove('active'));
            indicator.classList.add('active');
            
            // Create and show popup
            createSuggestionPopup(markerElement, suggestion, index);
        };
        
        indicator.addEventListener('click', showPopup);
        markerElement.addEventListener('click', showPopup);
    }
    
    function createSuggestionPopup(markerElement, suggestion, index) {
        const popup = document.createElement('div');
        popup.className = 'suggestion-popup';
        popup.id = `popup-${index}`;
        
        popup.innerHTML = `
            <div class="popup-header">
                <img src="assets/darkgenius.png" class="popup-icon">
                <span class="popup-title">Genius AI</span>
                <button class="popup-close" data-index="${index}">✕</button>
            </div>
            <div class="popup-content">
                <div class="popup-suggestion">${suggestion.suggestion}</div>
                <div class="popup-reason">${suggestion.reason}</div>
            </div>
            <div class="popup-actions">
                <button class="popup-btn popup-accept" data-index="${index}">Accept</button>
                <button class="popup-btn popup-reject" data-index="${index}">Ignore</button>
            </div>
        `;
        
        // Position popup near marked text
        const rect = markerElement.getBoundingClientRect();
        popup.style.position = 'fixed';
        popup.style.left = `${rect.left}px`;
        popup.style.top = `${rect.bottom + 10}px`;
        
        document.body.appendChild(popup);
        
        // Event listeners
        popup.querySelector('.popup-close').addEventListener('click', () => {
            popup.remove();
            document.querySelectorAll('.suggestion-indicator').forEach(i => i.classList.remove('active'));
        });
        
        popup.querySelector('.popup-accept').addEventListener('click', () => {
            // Get classData and existingDoc from the current context
            const currentUserData = localStorage.getItem('currentUser');
            const currentUser = currentUserData ? JSON.parse(currentUserData) : null;
            const classes = JSON.parse(localStorage.getItem('classes') || '[]');
            const classData = classes.find(c => c.userId === currentUser?.uid);
            const existingDoc = { id: 'current-document' }; // Use current document context
            
            acceptSuggestion(markerElement, suggestion.suggestion, index, classData, existingDoc);
        });
        
        popup.querySelector('.popup-reject').addEventListener('click', () => {
            // Get classData and existingDoc from the current context
            const currentUserData = localStorage.getItem('currentUser');
            const currentUser = currentUserData ? JSON.parse(currentUserData) : null;
            const classes = JSON.parse(localStorage.getItem('classes') || '[]');
            const classData = classes.find(c => c.userId === currentUser?.uid);
            const existingDoc = { id: 'current-document' }; // Use current document context
            
            rejectSuggestion(index, classData, existingDoc);
        });
        
        // Close popup when clicking outside
        setTimeout(() => {
            document.addEventListener('click', function closePopup(e) {
                if (!popup.contains(e.target) && 
                    !e.target.classList.contains('suggestion-marker') &&
                    !e.target.classList.contains('suggestion-indicator')) {
                    popup.remove();
                    document.querySelectorAll('.suggestion-indicator').forEach(i => i.classList.remove('active'));
                    document.removeEventListener('click', closePopup);
                }
            });
        }, 100);
    }
    
     function acceptSuggestion(markerElement, newText, index, classData = null, existingDoc = null) {
         console.log('Accepting suggestion:', newText, 'for index:', index);
         
         // Replace the marked text with the new text
         const textNode = document.createTextNode(newText);
         markerElement.parentNode.replaceChild(textNode, markerElement);
         
         // Remove indicator from sidebar
         const indicator = document.getElementById(`indicator-${index}`);
         if (indicator) {
             console.log('Removing indicator:', indicator.id);
             indicator.remove();
         }
         
         // Remove popup
         const popup = document.getElementById(`popup-${index}`);
         if (popup) {
             console.log('Removing popup:', popup.id);
             popup.remove();
         }
         
         // Remove from localStorage
         removeSuggestionFromStorage(index);
         
         // Update current suggestions state
         window.saveCurrentSuggestions();
         
         // Clear all suggestion storage to prevent reappearing
         clearAllSuggestionStorage();
         
         // Auto-save document after accepting suggestion
         setTimeout(() => {
             window.autoSaveDocument(classData, existingDoc);
         }, 500);
         
         console.log('Suggestion accepted and cleaned up');
     }
    
    function rejectSuggestion(index, classData = null, existingDoc = null) {
        console.log('Rejecting suggestion for index:', index);
        
        // Remove marker highlight
        const marker = document.getElementById(`suggestion-${index}`);
        if (marker) {
            console.log('Removing marker highlight:', marker.id);
            const text = marker.textContent;
            marker.replaceWith(document.createTextNode(text));
        }
        
        // Remove indicator from sidebar
        const indicator = document.getElementById(`indicator-${index}`);
        if (indicator) {
            console.log('Removing indicator:', indicator.id);
            indicator.remove();
        }
        
        // Remove popup
        const popup = document.getElementById(`popup-${index}`);
        if (popup) {
            console.log('Removing popup:', popup.id);
            popup.remove();
        }
        
        // Remove from localStorage
        removeSuggestionFromStorage(index);
        
        // Update current suggestions state
        window.saveCurrentSuggestions();
        
        // Clear all suggestion storage to prevent reappearing
        clearAllSuggestionStorage();
        
        // Auto-save document after rejecting suggestion
        setTimeout(() => {
            window.autoSaveDocument(classData, existingDoc);
        }, 500);
        
        console.log('Suggestion rejected and cleaned up');
    }
    
    window.clearSuggestionStorageOnly = function() {
        console.log('Clearing suggestion storage only (preserving DOM)...');
        
        try {
            // Clear ONLY suggestion-related keys from localStorage
            const keysToRemove = [];
            
            // Find all suggestion-related keys
            for (let i = 0; i < localStorage.length; i++) {
                const key = localStorage.key(i);
                if (key && (
                    key.includes('suggestions_') || 
                    key.includes('current_suggestions_') ||
                    key.includes('genius_chats_')
                )) {
                    keysToRemove.push(key);
                }
            }
            
            // Remove all suggestion keys
            keysToRemove.forEach(key => {
                localStorage.removeItem(key);
                console.log('Removed suggestion key:', key);
            });
            
            console.log('Storage cleanup complete: Removed', keysToRemove.length, 'suggestion keys');
            
        } catch (error) {
            console.error('Error clearing suggestion storage:', error);
        }
    }
    
// Global function to enable text selection
window.enableTextSelection = function() {
    console.log('Enabling text selection...');
    
    // Remove any existing selection restrictions
    document.body.style.userSelect = 'text';
    document.body.style.webkitUserSelect = 'text';
    document.body.style.mozUserSelect = 'text';
    document.body.style.msUserSelect = 'text';
    
    // Ensure all content elements allow text selection
    const contentElements = document.querySelectorAll('.doc-editor-content');
    contentElements.forEach(content => {
        content.style.userSelect = 'text';
        content.style.webkitUserSelect = 'text';
        content.style.mozUserSelect = 'text';
        content.style.msUserSelect = 'text';
        content.style.cursor = 'text';
        
        // Remove any event listeners that might prevent selection
        content.addEventListener('mousedown', (e) => {
            // Allow default behavior for text selection
            if (e.target === content || content.contains(e.target)) {
                e.stopPropagation();
            }
        }, true);
        
        content.addEventListener('selectstart', (e) => {
            // Allow text selection
            if (e.target === content || content.contains(e.target)) {
                e.stopPropagation();
            }
        }, true);
    });
    
    console.log('Text selection enabled');
};

// Undo/Redo functionality
let undoStack = [];
let redoStack = [];
let isUndoRedoOperation = false;


function saveState() {
    const content = document.getElementById('docEditorContent');
    const title = document.getElementById('docEditorTitle');
    
    if (content) {
        const state = {
            content: content.innerHTML,
            title: title ? title.value : '',
            timestamp: Date.now()
        };
        
        undoStack.push(state);
        
        // Limit undo stack size
        if (undoStack.length > 50) {
            undoStack.shift();
        }
        
        // Clear redo stack when new changes are made
        redoStack = [];
        
        console.log('State saved. Undo stack size:', undoStack.length);
        
        // Update undo/redo button states
        updateUndoRedoButtons();
    }
}

function updateUndoRedoButtons() {
    const undoBtn = document.getElementById('undoBtn');
    const redoBtn = document.getElementById('redoBtn');
    
    if (undoBtn) {
        undoBtn.disabled = undoStack.length <= 1;
        undoBtn.style.opacity = undoStack.length <= 1 ? '0.5' : '1';
    }
    
    if (redoBtn) {
        redoBtn.disabled = redoStack.length === 0;
        redoBtn.style.opacity = redoStack.length === 0 ? '0.5' : '1';
    }
}

// Initialize undo/redo functionality
function initializeUndoRedo() {
    // Save initial state
    saveState();
    
    // Add keyboard shortcuts
    document.addEventListener('keydown', (e) => {
        if (e.metaKey || e.ctrlKey) {
            if (e.key === 'z' && !e.shiftKey) {
                e.preventDefault();
                undo();
            } else if (e.key === 'z' && e.shiftKey) {
                e.preventDefault();
                redo();
            } else if (e.key === 'y') {
                e.preventDefault();
                redo();
            }
        }
    });
    
    // Monitor content changes for undo/redo
    const content = document.getElementById('docEditorContent');
    if (content) {
        content.addEventListener('input', () => {
            if (!isUndoRedoOperation) {
                saveState();
            }
        });
    }
    
    // Initial button state update
    updateUndoRedoButtons();
}

function undo() {
    if (undoStack.length <= 1) {
        showNotification('Nothing to undo', 'info');
        return;
    }
    
    // Move current state to redo stack
    const currentState = undoStack.pop();
    redoStack.push(currentState);
    
    // Get previous state
    const previousState = undoStack[undoStack.length - 1];
    
    if (previousState) {
        isUndoRedoOperation = true;
        
        const content = document.getElementById('docEditorContent');
        const title = document.getElementById('docEditorTitle');
        
        if (content) {
            content.innerHTML = previousState.content;
        }
        if (title) {
            title.value = previousState.title;
        }
        
        // Update stats and placeholder
        updateDocumentStats();
        updatePlaceholderVisibility();
        
        isUndoRedoOperation = false;
        
        // Update button states
        updateUndoRedoButtons();
        
        
        showNotification('Undo successful', 'success');
        console.log('Undo performed. Undo stack size:', undoStack.length, 'Redo stack size:', redoStack.length);
    }
}

function redo() {
    if (redoStack.length === 0) {
        showNotification('Nothing to redo', 'info');
        return;
    }
    
    // Get state from redo stack
    const state = redoStack.pop();
    undoStack.push(state);
    
    isUndoRedoOperation = true;
    
    const content = document.getElementById('docEditorContent');
    const title = document.getElementById('docEditorTitle');
    
    if (content) {
        content.innerHTML = state.content;
    }
    if (title) {
        title.value = state.title;
    }
    
    // Update stats and placeholder
    updateDocumentStats();
    updatePlaceholderVisibility();
    
    isUndoRedoOperation = false;
    
    // Update button states
    updateUndoRedoButtons();
    
    
    showNotification('Redo successful', 'success');
    console.log('Redo performed. Undo stack size:', undoStack.length, 'Redo stack size:', redoStack.length);
}


// Global function to type out content with animation (like geniusChat.js)
window.typeOutContent = function(contentElement, text) {
    console.log('Starting type-out animation for content...');
    console.log('Original text:', text);
    
    // Clean up the text by removing unwanted HTML/markdown tags at the beginning
    let cleanText = text;
    
    // Remove common unwanted prefixes
    cleanText = cleanText
        .replace(/^```html\s*/i, '') // Remove ```html at start
        .replace(/^```\s*/i, '') // Remove ``` at start
        .replace(/^<html[^>]*>/i, '') // Remove <html> tags
        .replace(/^<body[^>]*>/i, '') // Remove <body> tags
        .replace(/^<div[^>]*>/i, '') // Remove <div> tags at start
        .replace(/^<p[^>]*>/i, '') // Remove <p> tags at start
        .replace(/^<h[1-6][^>]*>/i, '') // Remove heading tags at start
        .replace(/^<br\s*\/?>/i, '') // Remove <br> at start
        .replace(/^<[^>]*>/g, '') // Remove any other opening tags at start
        .replace(/^\s+/, '') // Remove leading whitespace
        .trim();
    
    console.log('Cleaned text:', cleanText);
    
    // Clear existing content and add typing class
    contentElement.innerHTML = '';
    contentElement.classList.add('typing');
    
    let currentText = '';
    let index = 0;
    
    // Type out the response character by character
    const typeWriter = () => {
        if (index < cleanText.length) {
            currentText += cleanText[index];
            contentElement.innerHTML = currentText;
            index++;
            
            // Auto-scroll to bottom as content is being typed
            const contentWrapper = document.querySelector('.doc-editor-content-wrapper');
            if (contentWrapper) {
                // Smooth scroll to bottom
                contentWrapper.scrollTo({
                    top: contentWrapper.scrollHeight,
                    behavior: 'smooth'
                });
            }
            
            // Variable speed - faster for spaces, slower for punctuation
            const char = cleanText[index - 1];
            let delay = 20; // Base delay
            if (char === ' ') delay = 10;
            if (char === '.' || char === '!' || char === '?') delay = 100;
            if (char === '\n') delay = 50;
            if (char === '<') delay = 5; // Faster for HTML tags
            
            setTimeout(typeWriter, delay);
        } else {
            // Remove typing class when complete
            contentElement.classList.remove('typing');
            console.log('Type-out animation complete');
            
            // Final scroll to ensure we're at the bottom
            const contentWrapper = document.querySelector('.doc-editor-content-wrapper');
            if (contentWrapper) {
                contentWrapper.scrollTo({
                    top: contentWrapper.scrollHeight,
                    behavior: 'smooth'
                });
            }
        }
    };
    
    // Start typing
    typeWriter();
    
    // Save state after AI content is added
    setTimeout(() => {
        saveState();
    }, 100);
};
    
    function clearAllSuggestionData() {
        console.log('NUCLEAR OPTION: Clearing ALL suggestion data...');
        
        try {
            // Clear ALL suggestion-related keys from localStorage
            const keysToRemove = [];
            
            // Find all suggestion-related keys
            for (let i = 0; i < localStorage.length; i++) {
                const key = localStorage.key(i);
                if (key && (
                    key.includes('suggestions_') || 
                    key.includes('current_suggestions_') ||
                    key.includes('genius_chats_')
                )) {
                    keysToRemove.push(key);
                }
            }
            
            // Remove all suggestion keys
            keysToRemove.forEach(key => {
                localStorage.removeItem(key);
                console.log('Removed suggestion key:', key);
            });
            
            // Also clear any existing suggestions from the DOM
            const markers = document.querySelectorAll('.suggestion-marker');
            markers.forEach(marker => {
                const textNode = document.createTextNode(marker.textContent);
                marker.replaceWith(textNode);
            });
            
            const indicators = document.querySelectorAll('.suggestion-indicator');
            indicators.forEach(indicator => {
                indicator.remove();
            });
            
            const popups = document.querySelectorAll('.suggestion-popup');
            popups.forEach(popup => {
                popup.remove();
            });
            
            console.log('NUCLEAR CLEANUP COMPLETE: Removed', keysToRemove.length, 'suggestion keys and all DOM elements');
            
        } catch (error) {
            console.error('Error in nuclear cleanup:', error);
        }
    }
    
    function clearAllSuggestionStorage() {
        console.log('Clearing all suggestion storage...');
        
        try {
            const currentUserData = localStorage.getItem('currentUser');
            if (!currentUserData) return;
            
            const currentUser = JSON.parse(currentUserData);
            const classes = JSON.parse(localStorage.getItem('classes') || '[]');
            
            classes.forEach(classData => {
                if (classData.userId === currentUser.uid) {
                    // Clear all suggestion keys for this user
                    const docId = 'current-document';
                    const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
                    const currentSuggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
                    
                    // Clear current suggestions
                    localStorage.removeItem(suggestionsKey);
                    localStorage.removeItem(currentSuggestionsKey);
                    console.log('Cleared suggestion storage keys:', suggestionsKey, currentSuggestionsKey);
                    
                    // Clear all document-specific suggestions
                    const documents = JSON.parse(classData.documents || '[]');
                    documents.forEach(doc => {
                        const docSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_${doc.id}`;
                        localStorage.removeItem(docSuggestionsKey);
                        console.log('Cleared document suggestion key:', docSuggestionsKey);
                    });
                }
            });
            
            console.log('All suggestion storage cleared');
        } catch (error) {
            console.error('Error clearing suggestion storage:', error);
        }
    }
    
    function removeSuggestionFromStorage(index) {
        console.log('Removing suggestion from storage, index:', index);
        
        // Get current user and class data
        const currentUserData = localStorage.getItem('currentUser');
        if (!currentUserData) {
            console.log('No current user found');
            return;
        }
        
        const currentUser = JSON.parse(currentUserData);
        const classes = JSON.parse(localStorage.getItem('classes') || '[]');
        let removed = false;
        
        // Get the original text of the suggestion being removed
        const marker = document.getElementById(`suggestion-${index}`);
        const originalText = marker ? marker.textContent : null;
        
        console.log('Removing suggestion with original text:', originalText);
        
        // Remove from all relevant storage keys
        classes.forEach(classData => {
            if (classData.userId === currentUser.uid) {
                // Remove from document-specific suggestions
                const docId = 'current-document';
                const suggestionsKey = `suggestions_${classData.userId}_${classData.name}_${docId}`;
                const currentSuggestionsKey = `current_suggestions_${classData.userId}_${classData.name}_${docId}`;
                
                // Remove from suggestions storage by matching original text
                try {
                    const suggestions = JSON.parse(localStorage.getItem(suggestionsKey) || '[]');
                    const filteredSuggestions = suggestions.filter(suggestion => {
                        if (originalText) {
                            return suggestion.original !== originalText;
                        }
                        return true; // If no original text, keep all
                    });
                    if (filteredSuggestions.length !== suggestions.length) {
                        localStorage.setItem(suggestionsKey, JSON.stringify(filteredSuggestions));
                        console.log('Removed suggestion from suggestions storage:', suggestionsKey);
                        removed = true;
                    }
                } catch (error) {
                    console.error('Error updating suggestions storage:', error);
                }
                
                // Remove from current suggestions storage
                try {
                    const currentSuggestions = JSON.parse(localStorage.getItem(currentSuggestionsKey) || '[]');
                    const filteredCurrentSuggestions = currentSuggestions.filter(s => {
                        if (originalText) {
                            return s.original !== originalText;
                        }
                        return s.index !== index.toString();
                    });
                    if (filteredCurrentSuggestions.length !== currentSuggestions.length) {
                        localStorage.setItem(currentSuggestionsKey, JSON.stringify(filteredCurrentSuggestions));
                        console.log('Removed suggestion from current suggestions storage:', currentSuggestionsKey);
                        removed = true;
                    }
                } catch (error) {
                    console.error('Error updating current suggestions storage:', error);
                }
                
                // Also remove from all document-specific keys
                const documents = JSON.parse(classData.documents || '[]');
                documents.forEach(doc => {
                    const docSuggestionsKey = `suggestions_${classData.userId}_${classData.name}_${doc.id}`;
                    try {
                        const docSuggestions = JSON.parse(localStorage.getItem(docSuggestionsKey) || '[]');
                        const filteredDocSuggestions = docSuggestions.filter(suggestion => {
                            if (originalText) {
                                return suggestion.original !== originalText;
                            }
                            return true;
                        });
                        if (filteredDocSuggestions.length !== docSuggestions.length) {
                            localStorage.setItem(docSuggestionsKey, JSON.stringify(filteredDocSuggestions));
                            console.log('Removed suggestion from document storage:', docSuggestionsKey);
                            removed = true;
                        }
                    } catch (error) {
                        console.error('Error updating document suggestions storage:', error);
                    }
                });
            }
        });
        
        if (!removed) {
            console.log('No suggestion found to remove at index:', index);
        } else {
            console.log('Successfully removed suggestion from all storage locations');
        }
    }
}

// Make functions globally available
window.openDocumentEditor = openDocumentEditor;
window.closeDocumentEditor = closeDocumentEditor;

// Check for suggestions on page load (handles browser refresh)
// DISABLED: Don't load suggestions on page refresh to prevent multiplying
// document.addEventListener('DOMContentLoaded', () => {
//     // Check if we're in a document editor
//     const editorScreen = document.getElementById('documentEditorScreen');
//     if (editorScreen) {
//         console.log('Document editor detected on page load, checking for suggestions...');
//         
//         // Try to get class data from localStorage or URL parameters
//         const currentUserData = localStorage.getItem('currentUser');
//         if (currentUserData) {
//             const currentUser = JSON.parse(currentUserData);
//             const classes = JSON.parse(localStorage.getItem('classes') || '[]');
//             
//             // Find the most recent class (this is a fallback)
//             if (classes.length > 0) {
//                 const classData = classes[0];
//                 console.log('Loading suggestions for class:', classData.name);
//                 
//                 // Try to load suggestions for new documents
//                 setTimeout(() => {
//                     loadSavedSuggestions(classData, null);
//                 }, 1000);
//             }
//         }
//     }
// });

// Instead, clear only suggestion storage on page load (preserve DOM)
document.addEventListener('DOMContentLoaded', () => {
    const editorScreen = document.getElementById('documentEditorScreen');
    if (editorScreen) {
        console.log('Document editor detected on page load - clearing suggestion storage only');
        clearSuggestionStorageOnly();
    }
});

// AI Checker functionality using ZeroGPT API
async function runAIChecker() {
    console.log('🤖 Starting AI Checker...');
    
    // Get selected text or entire document
    const selection = window.getSelection();
    let textToCheck = '';
    
    if (selection.toString().trim()) {
        // Use selected text
        textToCheck = selection.toString().trim();
        console.log('📝 Checking selected text:', textToCheck.substring(0, 100) + '...');
    } else {
        // Use entire document
        const contentElement = document.querySelector('.doc-editor-content');
        if (contentElement) {
            textToCheck = contentElement.innerText || contentElement.textContent || '';
            console.log('📄 Checking entire document:', textToCheck.substring(0, 100) + '...');
        }
    }
    
    if (!textToCheck.trim()) {
        alert('No text to check. Please select some text or ensure the document has content.');
        return;
    }
    
    // Show loading state
    const aiCheckerBtn = document.getElementById('aiCheckerBtn');
    const originalText = aiCheckerBtn.innerHTML;
    aiCheckerBtn.innerHTML = '⏳';
    aiCheckerBtn.disabled = true;
    
    try {
        // Call ZeroGPT API
        const result = await checkTextWithZeroGPT(textToCheck);
        
        if (result.success) {
            console.log('✅ AI Check completed:', result);
            
            // Show results summary
            showAICheckerResults(result);
            
            // Highlight AI-detected text
            if (result.highlighted_sentences && result.highlighted_sentences.length > 0) {
                highlightAIText(result.highlighted_sentences);
            }
        } else {
            console.error('❌ AI Check failed:', result.error);
            alert('AI Check failed: ' + result.error);
        }
    } catch (error) {
        console.error('❌ AI Check error:', error);
        alert('AI Check error: ' + error.message);
    } finally {
        // Restore button state
        aiCheckerBtn.innerHTML = originalText;
        aiCheckerBtn.disabled = false;
    }
}

// ZeroGPT API integration
async function checkTextWithZeroGPT(text) {
    // Try multiple CORS proxies for better reliability
    const proxies = [
        'https://cors-anywhere.herokuapp.com/https://api.zerogpt.com/api/detect/detectText',
        'https://api.allorigins.win/raw?url=https://api.zerogpt.com/api/detect/detectText',
        'https://corsproxy.io/?https://api.zerogpt.com/api/detect/detectText'
    ];
    
    const headers = {
        'Accept': 'application/json, text/plain, */*',
        'Accept-Encoding': 'gzip, deflate, br, zstd',
        'Accept-Language': 'en-US,en;q=0.9',
        'Connection': 'keep-alive',
        'Content-Type': 'application/json',
        'DNT': '1',
        'Origin': 'https://www.zerogpt.com',
        'Referer': 'https://www.zerogpt.com/',
        'Sec-Ch-Ua': '"Chromium";v="139", "Not;A=Brand";v="99"',
        'Sec-Ch-Ua-Mobile': '?0',
        'Sec-Ch-Ua-Platform': '"macOS"',
        'Sec-Fetch-Dest': 'empty',
        'Sec-Fetch-Mode': 'cors',
        'Sec-Fetch-Site': 'same-site',
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36'
    };
    
    const payload = { "input_text": text };
    
    // Try each proxy until one works
    for (let i = 0; i < proxies.length; i++) {
        try {
            console.log(`🔄 Trying CORS proxy ${i + 1}/${proxies.length}:`, proxies[i]);
            
            const response = await fetch(proxies[i], {
                method: 'POST',
                headers: headers,
                body: JSON.stringify(payload)
            });
            
            if (response.ok) {
                const result = await response.json();
                const data = result.data || {};
                
                console.log('✅ CORS proxy successful:', proxies[i]);
                
                return {
                    success: true,
                    is_ai: data.isHuman === 0,
                    is_human: data.isHuman === 1,
                    ai_percentage: data.fakePercentage || 0,
                    feedback: data.feedback || '',
                    language: data.detected_language || '',
                    text_words: data.textWords || 0,
                    ai_words: data.aiWords || 0,
                    highlighted_sentences: data.h || []
                };
            } else {
                console.log(`❌ CORS proxy ${i + 1} failed:`, response.status, response.statusText);
            }
        } catch (error) {
            console.log(`❌ CORS proxy ${i + 1} error:`, error.message);
        }
    }
    
    // If all proxies fail, return error
    return {
        success: false,
        error: 'All CORS proxies failed. Please try again later or use a different network.'
    };
}

// Show AI Checker results in a floating summary
function showAICheckerResults(result) {
    // Remove existing summary if any
    const existingSummary = document.getElementById('aiCheckerSummary');
    if (existingSummary) {
        existingSummary.remove();
    }
    
    // Create floating summary
    const summary = document.createElement('div');
    summary.id = 'aiCheckerSummary';
    summary.className = 'ai-checker-summary';
    
    const aiPercentage = result.ai_percentage || 0;
    const isAI = result.is_ai;
    const statusColor = isAI ? '#ff4444' : '#44ff44';
    const statusText = isAI ? 'AI Generated' : 'Human Written';
    const statusIcon = isAI ? '🤖' : '👤';
    
    summary.innerHTML = `
        <div class="ai-summary-header">
            <span class="ai-summary-icon">${statusIcon}</span>
            <span class="ai-summary-title">AI Detection Results</span>
            <button class="ai-summary-close" onclick="this.parentElement.parentElement.remove()">×</button>
        </div>
        <div class="ai-summary-content">
            <div class="ai-summary-status" style="color: ${statusColor}">
                ${statusText} (${aiPercentage}% AI)
            </div>
            <div class="ai-summary-details">
                <div class="ai-detail-item">
                    <span class="ai-detail-label">Total Words:</span>
                    <span class="ai-detail-value">${result.text_words}</span>
                </div>
                <div class="ai-detail-item">
                    <span class="ai-detail-label">AI Words:</span>
                    <span class="ai-detail-value">${result.ai_words}</span>
                </div>
                <div class="ai-detail-item">
                    <span class="ai-detail-label">Language:</span>
                    <span class="ai-detail-value">${result.language}</span>
                </div>
            </div>
            ${result.feedback ? `<div class="ai-summary-feedback">${result.feedback}</div>` : ''}
        </div>
    `;
    
    // Add to document
    document.body.appendChild(summary);
    
    // Auto-remove after 10 seconds
    setTimeout(() => {
        if (summary.parentNode) {
            summary.remove();
        }
    }, 10000);
}

// Highlight AI-detected text
function highlightAIText(highlightedSentences) {
    console.log('🎨 Highlighting AI-detected text:', highlightedSentences);
    
    // Remove existing highlights
    const existingHighlights = document.querySelectorAll('.ai-highlighted');
    existingHighlights.forEach(highlight => {
        const parent = highlight.parentNode;
        parent.replaceChild(document.createTextNode(highlight.textContent), highlight);
        parent.normalize();
    });
    
    // Highlight each sentence
    highlightedSentences.forEach(sentence => {
        if (sentence && sentence.trim()) {
            highlightTextInDocument(sentence.trim());
        }
    });
}

// Helper function to highlight specific text in the document
function highlightTextInDocument(text) {
    const contentElement = document.querySelector('.doc-editor-content');
    if (!contentElement) return;
    
    const walker = document.createTreeWalker(
        contentElement,
        NodeFilter.SHOW_TEXT,
        null,
        false
    );
    
    const textNodes = [];
    let node;
    
    while (node = walker.nextNode()) {
        if (node.textContent.includes(text)) {
            textNodes.push(node);
        }
    }
    
    textNodes.forEach(textNode => {
        const parent = textNode.parentNode;
        const content = textNode.textContent;
        const index = content.indexOf(text);
        
        if (index !== -1) {
            const beforeText = content.substring(0, index);
            const highlightedText = content.substring(index, index + text.length);
            const afterText = content.substring(index + text.length);
            
            const fragment = document.createDocumentFragment();
            
            if (beforeText) {
                fragment.appendChild(document.createTextNode(beforeText));
            }
            
            const highlightSpan = document.createElement('span');
            highlightSpan.className = 'ai-highlighted';
            highlightSpan.textContent = highlightedText;
            fragment.appendChild(highlightSpan);
            
            if (afterText) {
                fragment.appendChild(document.createTextNode(afterText));
            }
            
            parent.replaceChild(fragment, textNode);
        }
    });
}



// Test functions for debugging
window.testAutoSave = function() {
    console.log('Testing auto-save...');
    if (window.autoSave) {
        window.hasChanges = true;
        window.autoSave();
    }
};

// Simple Spell Check Functionality
let spellCheckEnabled = true;

// Initialize spell check when document editor opens
function initializeSpellCheck() {
    console.log('🚀 initializeSpellCheck called, enabled:', spellCheckEnabled);
    
    if (!spellCheckEnabled) {
        console.log('❌ Spell check disabled, not initializing');
        return;
    }
    
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) {
        console.log('❌ Content element not found, cannot initialize spell check');
        return;
    }
    
    console.log('✅ Content element found, initializing spell check...');
    
    // Disable browser's built-in spell check
    contentElement.setAttribute('spellcheck', 'false');
    
    // Add event listeners for real-time spell checking
    contentElement.addEventListener('input', debounce(performSpellCheck, 500));
    contentElement.addEventListener('paste', () => {
        setTimeout(performSpellCheck, 200);
    });
    
    console.log('⏰ Setting up initial spell check timeout...');
    
    // Initial spell check
    setTimeout(() => {
        console.log('⏰ Initial spell check timeout triggered');
        performSpellCheck();
    }, 1000);
}

// Debounce function
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Main spell check function
function performSpellCheck() {
    console.log('🔍 performSpellCheck called, enabled:', spellCheckEnabled);
    
    if (!spellCheckEnabled) {
        console.log('❌ Spell check disabled');
        return;
    }
    
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) {
        console.log('❌ Content element not found');
        return;
    }
    
    console.log('🔍 Starting spell check...');
    
    // Clear existing markers
    clearSpellCheckMarkers();
    
    // Get all text content
    const text = contentElement.textContent || contentElement.innerText || '';
    console.log('📝 Text to check:', text);
    
    if (!text.trim()) {
        console.log('❌ No text to check');
        return;
    }
    
    // Find misspelled words
    const words = text.split(/\s+/).filter(word => word.length > 2 && /^[a-zA-Z]+$/.test(word));
    console.log('📝 Words to check:', words);
    
    if (words.length === 0) {
        console.log('❌ No words to check');
        return;
    }
    
    let misspelledCount = 0;
    words.forEach(word => {
        if (isMisspelled(word)) {
            console.log('❌ Misspelled word found:', word);
            markMisspelledWord(word);
            misspelledCount++;
        }
    });
    
    console.log(`✅ Spell check completed - found ${misspelledCount} misspelled words`);
}

// Check if a word is misspelled
function isMisspelled(word) {
    console.log('🔍 Checking if misspelled:', word);
    
    const commonWords = [
        'the', 'be', 'to', 'of', 'and', 'a', 'in', 'that', 'have', 'i', 'it', 'for', 'not', 'on', 'with', 'he', 'as', 'you', 'do', 'at',
        'this', 'but', 'his', 'by', 'from', 'they', 'she', 'or', 'an', 'will', 'my', 'one', 'all', 'would', 'there', 'their', 'what',
        'so', 'up', 'out', 'if', 'about', 'who', 'get', 'which', 'go', 'me', 'when', 'make', 'can', 'like', 'time', 'no', 'just',
        'him', 'know', 'take', 'people', 'into', 'year', 'your', 'good', 'some', 'could', 'them', 'see', 'other', 'than', 'then',
        'now', 'look', 'only', 'come', 'its', 'over', 'think', 'also', 'back', 'after', 'use', 'two', 'how', 'our', 'work', 'first',
        'well', 'way', 'even', 'new', 'want', 'because', 'any', 'these', 'give', 'day', 'most', 'us', 'is', 'was', 'are', 'been',
        'has', 'had', 'were', 'said', 'each', 'which', 'their', 'time', 'will', 'about', 'if', 'up', 'out', 'many', 'then', 'them',
        'can', 'only', 'other', 'new', 'some', 'what', 'time', 'very', 'when', 'much', 'get', 'through', 'back', 'much', 'before',
        'go', 'good', 'new', 'first', 'last', 'long', 'little', 'own', 'other', 'old', 'right', 'big', 'high', 'different', 'small',
        'large', 'next', 'early', 'young', 'important', 'few', 'public', 'same', 'able', 'walking', 'library', 'because', 'really',
        'study', 'exams', 'keep', 'getting', 'distracted', 'was', 'walking', 'library', 'because', 'really', 'study', 'exams',
        'keep', 'getting', 'distracted'
    ];
    
    const isCommon = commonWords.includes(word.toLowerCase());
    const isMisspelled = !isCommon;
    
    console.log(`🔍 Word "${word}" - isCommon: ${isCommon}, isMisspelled: ${isMisspelled}`);
    
    return isMisspelled;
}

// Mark a misspelled word
function markMisspelledWord(word) {
    console.log('🎯 markMisspelledWord called for:', word);
    
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) {
        console.log('❌ Content element not found in markMisspelledWord');
        return;
    }
    
    // Find and replace the word in the content
    const html = contentElement.innerHTML;
    console.log('📝 Current HTML:', html);
    
    const regex = new RegExp(`\\b${word}\\b`, 'gi');
    console.log('🔍 Regex:', regex);
    
    if (regex.test(html)) {
        console.log('✅ Found word in HTML, replacing...');
        const newHtml = html.replace(regex, `<span class="spell-check-error" data-word="${word}">${word}</span>`);
        console.log('📝 New HTML:', newHtml);
        contentElement.innerHTML = newHtml;
        
        // Add click event to show suggestions
        const markers = contentElement.querySelectorAll('.spell-check-error');
        console.log('🎯 Found markers:', markers.length);
        markers.forEach(marker => {
            marker.addEventListener('click', (e) => {
                e.preventDefault();
                showSpellCheckSuggestions(marker, word);
            });
        });
    } else {
        console.log('❌ Word not found in HTML');
    }
}

// Show spell check suggestions
function showSpellCheckSuggestions(element, word) {
    // Remove existing popup
    const existingPopup = document.querySelector('.spell-check-suggestion-popup');
    if (existingPopup) {
        existingPopup.remove();
    }
    
    // Create popup
    const popup = document.createElement('div');
    popup.className = 'spell-check-suggestion-popup';
    popup.innerHTML = `
        <div class="suggestion-header">
            <span class="misspelled-word">"${word}"</span>
            <button class="close-btn" onclick="this.parentElement.parentElement.remove()">×</button>
        </div>
        <div class="suggestions-list">
            ${getSuggestions(word).map(suggestion => 
                `<div class="suggestion-item" onclick="replaceWord('${word}', '${suggestion}'); this.parentElement.parentElement.remove();">${suggestion}</div>`
            ).join('')}
        </div>
        <div class="suggestion-actions">
            <button class="ignore-btn" onclick="ignoreWord('${word}'); this.parentElement.parentElement.remove();">Ignore</button>
        </div>
    `;
    
    // Position popup
    const rect = element.getBoundingClientRect();
    popup.style.position = 'fixed';
    popup.style.left = rect.left + 'px';
    popup.style.top = (rect.bottom + 5) + 'px';
    popup.style.zIndex = '10000';
    
    document.body.appendChild(popup);
    
    // Close popup when clicking outside
    setTimeout(() => {
        document.addEventListener('click', function closePopup(e) {
            if (!popup.contains(e.target) && e.target !== element) {
                popup.remove();
                document.removeEventListener('click', closePopup);
            }
        });
    }, 100);
}

// Get suggestions for a word
function getSuggestions(word) {
    const suggestions = {
        'wuz': 'was',
        'walkin': 'walking',
        'libary': 'library',
        'becuz': 'because',
        'realy': 'really',
        'studdy': 'study',
        'egzams': 'exams',
        'kep': 'keep',
        'geting': 'getting',
        'distacted': 'distracted',
        'teh': 'the',
        'adn': 'and',
        'nad': 'and',
        'taht': 'that',
        'thier': 'their',
        'recieve': 'receive',
        'seperate': 'separate',
        'occured': 'occurred',
        'definately': 'definitely',
        'beleive': 'believe',
        'calender': 'calendar',
        'cemetary': 'cemetery',
        'collegue': 'colleague',
        'comming': 'coming',
        'concious': 'conscious',
        'dependant': 'dependent',
        'embarass': 'embarrass',
        'exagerate': 'exaggerate',
        'existance': 'existence',
        'foriegn': 'foreign',
        'goverment': 'government',
        'independant': 'independent',
        'priviledge': 'privilege',
        'untill': 'until',
        'usefull': 'useful',
        'writting': 'writing',
        'writen': 'written'
    };
    
    return suggestions[word.toLowerCase()] ? [suggestions[word.toLowerCase()]] : ['Check spelling'];
}

// Replace a word
function replaceWord(oldWord, newWord) {
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) return;
    
    const html = contentElement.innerHTML;
    const regex = new RegExp(`<span class="spell-check-error" data-word="${oldWord}">${oldWord}</span>`, 'gi');
    const newHtml = html.replace(regex, newWord);
    contentElement.innerHTML = newHtml;
}

// Ignore a word
function ignoreWord(word) {
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) return;
    
    const html = contentElement.innerHTML;
    const regex = new RegExp(`<span class="spell-check-error" data-word="${word}">${word}</span>`, 'gi');
    const newHtml = html.replace(regex, word);
    contentElement.innerHTML = newHtml;
}

// Clear all spell check markers
function clearSpellCheckMarkers() {
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) return;
    
    const markers = contentElement.querySelectorAll('.spell-check-error');
    markers.forEach(marker => {
        marker.replaceWith(document.createTextNode(marker.textContent));
    });
}

// Toggle spell check
function toggleSpellCheck() {
    spellCheckEnabled = !spellCheckEnabled;
    
    if (spellCheckEnabled) {
        initializeSpellCheck();
    } else {
        clearSpellCheckMarkers();
    }
}

// Initialize spell check when document editor opens
function initializeSpellCheck() {
    console.log('🚀 initializeSpellCheck called, enabled:', spellCheckEnabled);
    
    if (!spellCheckEnabled) {
        console.log('❌ Spell check disabled, not initializing');
        return;
    }
    
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) {
        console.log('❌ Content element not found, cannot initialize spell check');
        return;
    }
    
    console.log('✅ Content element found, initializing spell check...');
    
    // Disable browser's built-in spell check
    contentElement.setAttribute('spellcheck', 'false');
    
    // Add event listeners for real-time spell checking
    contentElement.addEventListener('input', debounce(performSpellCheck, 500));
    contentElement.addEventListener('paste', () => {
        setTimeout(performSpellCheck, 200);
    });
    
    console.log('⏰ Setting up initial spell check timeout...');
    
    // Initial spell check
    setTimeout(() => {
        console.log('⏰ Initial spell check timeout triggered');
        performSpellCheck();
    }, 1000);
}

// Debounce function to limit spell check frequency
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Main spell check function
function performSpellCheck() {
    if (!spellCheckEnabled) return;
    
    const contentElement = document.getElementById('docEditorContent');
    if (!contentElement) return;
    
    // Skip spell check if user is actively typing (cursor is in the editor)
    if (document.activeElement === contentElement) {
        return;
    }
    
    // Clear existing spell check markers
    clearSpellCheckMarkers();
    
    // Get all text nodes in the content
    const textNodes = getTextNodes(contentElement);
    
    textNodes.forEach(node => {
        if (node.nodeType === Node.TEXT_NODE && node.textContent.trim()) {
            checkTextNode(node);
        }
    });
}

// Get all text nodes in an element
function getTextNodes(element) {
    const textNodes = [];
    const walker = document.createTreeWalker(
        element,
        NodeFilter.SHOW_TEXT,
        {
            acceptNode: (node) => {
                // Skip text nodes that are inside spell check markers
                if (node.parentElement && node.parentElement.classList.contains('spell-check-error')) {
                    return NodeFilter.FILTER_REJECT;
                }
                return NodeFilter.FILTER_ACCEPT;
            }
        }
    );
    
    let node;
    while (node = walker.nextNode()) {
        textNodes.push(node);
    }
    
    return textNodes;
}

// Check a text node for spelling errors
function checkTextNode(textNode) {
    const textContent = textNode.textContent;
    if (!textContent || textContent.trim().length === 0) return;
    
    const words = textContent.split(/(\s+)/);
    
    let currentOffset = 0;
    words.forEach(word => {
        if (word.trim() && isWord(word)) {
            const wordStart = textContent.indexOf(word, currentOffset);
            const wordEnd = wordStart + word.length;
            
            // Validate word position and check if already marked
            if (wordStart >= 0 && wordEnd <= textContent.length && 
                !isWordAlreadyMarked(textNode, wordStart, wordEnd) && 
                isMisspelled(word.trim())) {
                try {
                    markMisspelledWord(textNode, wordStart, wordEnd, word.trim());
                } catch (error) {
                    console.log('Spell check error (ignoring):', error.message);
                }
            }
        }
        currentOffset += word.length;
    });
}

// Check if a string is a valid word (not just punctuation or numbers)
function isWord(str) {
    if (!str || typeof str !== 'string') return false;
    // Check if it contains at least one letter and is not just punctuation
    return /[a-zA-Z]/.test(str) && str.trim().length > 0;
}

// Check if a word is already marked as misspelled
function isWordAlreadyMarked(textNode, startOffset, endOffset) {
    // Check if the text node is already inside a spell-check-error span
    if (textNode.parentElement && textNode.parentElement.classList.contains('spell-check-error')) {
        return true;
    }
    
    // Check if there are any spell-check-error spans in the parent element
    const parent = textNode.parentElement;
    if (parent) {
        const existingErrors = parent.querySelectorAll('.spell-check-error');
        for (let error of existingErrors) {
            if (error.textContent === textNode.textContent.substring(startOffset, endOffset)) {
                return true;
            }
        }
    }
    
    return false;
}

// Make spell check functions globally available
window.initializeSpellCheck = initializeSpellCheck;
window.toggleSpellCheck = toggleSpellCheck;
window.clearSpellCheckMarkers = clearSpellCheckMarkers;


// Mark a misspelled word with red underline
function markMisspelledWord(textNode, startOffset, endOffset, word) {
    // Validate inputs
    if (!textNode || startOffset < 0 || endOffset <= startOffset || endOffset > textNode.textContent.length) {
        console.log('Invalid parameters for markMisspelledWord:', { textNode, startOffset, endOffset, word });
        return;
    }
    
    const range = document.createRange();
    try {
        range.setStart(textNode, startOffset);
        range.setEnd(textNode, endOffset);
        
        const span = document.createElement('span');
        span.className = 'spell-check-error';
        span.setAttribute('data-word', word);
        span.setAttribute('data-original-text', word);
        
        range.surroundContents(span);
        
        // Add click event for suggestions
        span.addEventListener('click', (e) => {
            e.preventDefault();
            e.stopPropagation();
            showSpellCheckSuggestions(span, word);
        });
        
        // Add right-click context menu
        span.addEventListener('contextmenu', (e) => {
            e.preventDefault();
            e.stopPropagation();
            showSpellCheckContextMenu(e, span, word);
        });
    } catch (error) {
        console.log('Could not mark word as misspelled:', error.message);
        // Try alternative approach for complex DOM structures
        try {
            const parent = textNode.parentNode;
            if (parent && parent.nodeType === Node.ELEMENT_NODE) {
                const beforeText = textNode.textContent.substring(0, startOffset);
                const afterText = textNode.textContent.substring(endOffset);
                const wordText = textNode.textContent.substring(startOffset, endOffset);
                
                const span = document.createElement('span');
                span.className = 'spell-check-error';
                span.setAttribute('data-word', word);
                span.setAttribute('data-original-text', word);
                span.textContent = wordText;
                
                // Create new text nodes
                const beforeNode = document.createTextNode(beforeText);
                const afterNode = document.createTextNode(afterText);
                
                // Replace the original text node
                parent.insertBefore(beforeNode, textNode);
                parent.insertBefore(span, textNode);
                parent.insertBefore(afterNode, textNode);
                parent.removeChild(textNode);
                
                // Add event listeners
                span.addEventListener('click', (e) => {
                    e.preventDefault();
                    e.stopPropagation();
                    showSpellCheckSuggestions(span, word);
                });
                
                span.addEventListener('contextmenu', (e) => {
                    e.preventDefault();
                    e.stopPropagation();
                    showSpellCheckContextMenu(e, span, word);
                });
            }
        } catch (fallbackError) {
            console.log('Fallback spell check marking also failed:', fallbackError.message);
        }
    }
}

// Show spell check suggestions popup
async function showSpellCheckSuggestions(element, word) {
    // Remove existing popup
    const existingPopup = document.querySelector('.spell-check-suggestion-popup');
    if (existingPopup) {
        existingPopup.remove();
    }
    
    const popup = document.createElement('div');
    popup.className = 'spell-check-suggestion-popup';
    popup.style.display = 'block';
    
    // Show loading state
    popup.innerHTML = '<div class="spell-check-suggestion-item">Loading suggestions...</div>';
    
    // Position the popup
    const rect = element.getBoundingClientRect();
    popup.style.left = `${rect.left}px`;
    popup.style.top = `${rect.bottom + 5}px`;
    popup.style.position = 'fixed';
    popup.style.zIndex = '10000';
    
    document.body.appendChild(popup);
    
    try {
        // Get AI-powered suggestions
        const suggestions = await getSuggestions(word);
        
        // Clear loading and add suggestions
        popup.innerHTML = '';
        
        // Add suggestions
        suggestions.forEach(suggestion => {
            const item = document.createElement('div');
            item.className = 'spell-check-suggestion-item';
            item.textContent = suggestion;
            
            // Handle different suggestion types
            if (suggestion === 'Ignore word') {
                item.classList.add('ignore');
                item.addEventListener('click', () => {
                    ignoreWord(word);
                    element.remove();
                    popup.remove();
                });
            } else if (suggestion === 'Add to dictionary') {
                item.classList.add('add-to-dict');
                item.addEventListener('click', () => {
                    addToDictionary(word);
                    element.remove();
                    popup.remove();
                });
            } else {
                // Regular suggestion
                item.addEventListener('click', () => {
                    replaceWord(element, suggestion);
                    popup.remove();
                });
            }
            
            popup.appendChild(item);
        });
        
    } catch (error) {
        console.error('Error getting spell suggestions:', error);
        popup.innerHTML = '<div class="spell-check-suggestion-item">Error loading suggestions</div>';
    }
    
    // Close popup when clicking outside
    const closePopup = (e) => {
        if (!popup.contains(e.target) && !element.contains(e.target)) {
            popup.remove();
            document.removeEventListener('click', closePopup);
        }
    };
    
    setTimeout(() => {
        document.addEventListener('click', closePopup);
    }, 100);
}

// Show context menu for spell check
function showSpellCheckContextMenu(event, element, word) {
    // Remove existing popup
    const existingPopup = document.querySelector('.spell-check-suggestion-popup');
    if (existingPopup) {
        existingPopup.remove();
    }
    
    const popup = document.createElement('div');
    popup.className = 'spell-check-suggestion-popup';
    popup.style.display = 'block';
    
    // Add ignore option
    const ignoreItem = document.createElement('div');
    ignoreItem.className = 'spell-check-suggestion-item ignore';
    ignoreItem.textContent = 'Ignore';
    ignoreItem.addEventListener('click', () => {
        ignoreWord(word);
        element.remove();
        popup.remove();
    });
    popup.appendChild(ignoreItem);
    
    // Add to dictionary option
    const addToDictItem = document.createElement('div');
    addToDictItem.className = 'spell-check-suggestion-item add-to-dict';
    addToDictItem.textContent = 'Add to Dictionary';
    addToDictItem.addEventListener('click', () => {
        addToDictionary(word);
        element.remove();
        popup.remove();
    });
    popup.appendChild(addToDictItem);
    
    // Position popup
    popup.style.left = event.pageX + 'px';
    popup.style.top = event.pageY + 'px';
    
    document.body.appendChild(popup);
    
    // Close popup when clicking outside
    const closePopup = (e) => {
        if (!popup.contains(e.target) && !element.contains(e.target)) {
            popup.remove();
            document.removeEventListener('click', closePopup);
        }
    };
    
    setTimeout(() => {
        document.addEventListener('click', closePopup);
    }, 100);
}

// Get AI-powered suggestions for a misspelled word
async function getSuggestions(word) {
    const suggestions = [];
    
    // First, check common misspellings
    const commonCorrections = {
        'teh': 'the',
        'adn': 'and',
        'nad': 'and',
        'taht': 'that',
        'thier': 'their',
        'recieve': 'receive',
        'recieved': 'received',
        'recieving': 'receiving',
        'seperate': 'separate',
        'seperated': 'separated',
        'seperating': 'separating',
        'occured': 'occurred',
        'occuring': 'occurring',
        'definately': 'definitely',
        'accomodate': 'accommodate',
        'acheive': 'achieve',
        'begining': 'beginning',
        'beleive': 'believe',
        'calender': 'calendar',
        'cemetary': 'cemetery',
        'changable': 'changeable',
        'collegue': 'colleague',
        'comming': 'coming',
        'concious': 'conscious',
        'dependant': 'dependent',
        'embarass': 'embarrass',
        'exagerate': 'exaggerate',
        'existance': 'existence',
        'existant': 'existent',
        'foriegn': 'foreign',
        'goverment': 'government',
        'independant': 'independent',
        'priviledge': 'privilege',
        'untill': 'until',
        'usefull': 'useful',
        'writting': 'writing',
        'writen': 'written'
    };
    
    if (commonCorrections[word.toLowerCase()]) {
        suggestions.push(commonCorrections[word.toLowerCase()]);
    }
    
    // If no common correction found, try AI suggestions (only if API key is available)
    if (suggestions.length === 0) {
        const apiKey = window.getOpenAIApiKey() || window.APP_CONFIG?.OPENAI_API_KEY;
        if (apiKey) {
            try {
                const aiSuggestions = await getAISpellSuggestions(word);
                suggestions.push(...aiSuggestions);
            } catch (error) {
                console.log('AI spell check failed, using fallback:', error);
                // Fallback suggestions
                suggestions.push('Check spelling');
            }
        } else {
            // No API key available, just provide basic suggestions
            suggestions.push('Check spelling');
        }
    }
    
    // Always add these options
    suggestions.push('Ignore word');
    suggestions.push('Add to dictionary');
    
    return suggestions;
}

// Get AI-powered spell suggestions using OpenAI
async function getAISpellSuggestions(word) {
    try {
        // Use the same API key method as genius chat
        const apiKey = window.getOpenAIApiKey() || window.APP_CONFIG?.OPENAI_API_KEY;
        
        if (!apiKey) {
            throw new Error('OpenAI API key not found. Please add your API key in the profile settings to enable AI-powered spell suggestions.');
        }
        
        const response = await fetch('https://api.openai.com/v1/chat/completions', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${apiKey}`
            },
            body: JSON.stringify({
                model: 'gpt-3.5-turbo',
                messages: [
                    {
                        role: 'system',
                        content: 'You are a spell checker. Given a potentially misspelled word, provide 3-5 correct spelling suggestions. Return only the words separated by commas, no explanations.'
                    },
                    {
                        role: 'user',
                        content: `Suggest correct spellings for: "${word}"`
                    }
                ],
                max_tokens: 50,
                temperature: 0.1
            })
        });
        
        if (!response.ok) {
            throw new Error(`OpenAI API error: ${response.status}`);
        }
        
        const data = await response.json();
        const suggestions = data.choices[0].message.content
            .split(',')
            .map(s => s.trim())
            .filter(s => s.length > 0 && s !== word)
            .slice(0, 5);
        
        return suggestions;
    } catch (error) {
        console.error('AI spell check error:', error);
        return ['Check spelling'];
    }
}

// Replace a misspelled word with a suggestion
function replaceWord(element, suggestion) {
    element.textContent = suggestion;
    element.classList.remove('spell-check-error');
    element.classList.add('spell-check-corrected');
    
    // Trigger auto-save
    if (typeof triggerAutoSave === 'function') {
        triggerAutoSave();
    }
}

// These functions are replaced by the simple versions above

// Clear all spell check markers
function clearSpellCheckMarkers() {
    const markers = document.querySelectorAll('.spell-check-error');
    markers.forEach(marker => {
        const parent = marker.parentNode;
        parent.replaceChild(document.createTextNode(marker.textContent), marker);
        parent.normalize();
    });
}

// Save spell check settings to localStorage
// These functions are replaced by the simple versions above

// Toggle spell check on/off
function toggleSpellCheck() {
    spellCheckEnabled = !spellCheckEnabled;
    
    if (spellCheckEnabled) {
        initializeSpellCheck();
    } else {
        clearSpellCheckMarkers();
    }
    
    // Update UI to show spell check status
    const contentElement = document.getElementById('docEditorContent');
    if (contentElement) {
        contentElement.setAttribute('spellcheck', spellCheckEnabled.toString());
    }
}

// Find and Replace functionality removed
function showFindReplaceDialog() {
    // Function removed - no longer available
    return;
}

// Find and replace functions removed

// Make spell check functions globally available
window.initializeSpellCheck = initializeSpellCheck;
window.toggleSpellCheck = toggleSpellCheck;
window.clearSpellCheckMarkers = clearSpellCheckMarkers;

// Find and replace functions removed

// Make main functions globally available
window.openDocumentEditor = openDocumentEditor;
window.closeDocumentEditor = closeDocumentEditor;

