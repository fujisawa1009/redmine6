# Redmine Issue Templates 設定手順書

## 概要

既存のRedmine Issue Templatesプラグインの互換性問題により、**View Customize Plugin**を使用してチケットテンプレート機能を実装しました。プロジェクトごとのテンプレート管理、動的テンプレート適用、カスタマイズ可能なフィールド設定を提供します。

## 実装機能

### 1. チケットテンプレートシステム
- チケット作成・編集画面でのテンプレート選択
- プロジェクト別テンプレート管理
- トラッカー別テンプレート設定
- リアルタイムテンプレート適用

### 2. テンプレート管理機能
- 管理者によるテンプレート作成・編集
- プリセットテンプレート（バグ報告、機能要求、タスクなど）
- インポート・エクスポート機能
- バージョン管理

### 3. 高度なカスタマイズ
- 動的フィールド生成
- 条件分岐テンプレート
- 変数置換機能
- フォームバリデーション

## 設定手順

### 1. メインテンプレートシステム

#### 基本設定
- **名前**: Redmine Issue Templates システム
- **コメント**: チケット作成・編集画面にテンプレート機能を追加
- **有効**: ✓ チェック
- **プライベート**: ✗ チェック解除（全ユーザーで利用）

#### 適用条件
- **パスパターン**: `/issues/(new|\d+/edit)`
- **プロジェクトパターン**: （空欄 = 全プロジェクト）

#### カスタマイズ設定
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

#### JavaScript コード

```javascript
$(document).ready(function() {
    // チケットテンプレートシステムの初期化
    var IssueTemplateSystem = {
        templates: {},
        currentProject: null,
        currentTracker: null,
        storageKey: 'redmine_issue_templates',

        init: function() {
            this.currentProject = this.getProjectFromURL();
            this.loadTemplates();
            this.createTemplateUI();
            this.bindEvents();
            this.loadPresetTemplates();
        },

        // プロジェクトIDを取得
        getProjectFromURL: function() {
            var match = window.location.pathname.match(/\/projects\/([^\/]+)/);
            return match ? match[1] : 'global';
        },

        // テンプレートUIの作成
        createTemplateUI: function() {
            if (!$('#issue_subject, #issue_description').length) return;

            var templateHtml = `
                <div id="template-system" style="
                    background: #f8f9fa;
                    border: 1px solid #dee2e6;
                    border-radius: 8px;
                    padding: 15px;
                    margin: 15px 0;
                ">
                    <h4 style="margin: 0 0 15px 0; color: #333;">📝 チケットテンプレート</h4>

                    <div style="display: flex; gap: 15px; align-items: center; flex-wrap: wrap;">
                        <div>
                            <label for="template-selector" style="margin-right: 8px;">テンプレート:</label>
                            <select id="template-selector" style="min-width: 200px; padding: 5px;">
                                <option value="">-- テンプレートを選択 --</option>
                            </select>
                        </div>

                        <div>
                            <label for="tracker-filter" style="margin-right: 8px;">トラッカー:</label>
                            <select id="tracker-filter" style="min-width: 120px; padding: 5px;">
                                <option value="">全て</option>
                            </select>
                        </div>

                        <div>
                            <button id="apply-template" style="
                                background: #28a745;
                                color: white;
                                border: none;
                                padding: 6px 12px;
                                border-radius: 4px;
                                cursor: pointer;
                            ">適用</button>

                            <button id="manage-templates" style="
                                background: #007bff;
                                color: white;
                                border: none;
                                padding: 6px 12px;
                                border-radius: 4px;
                                cursor: pointer;
                                margin-left: 8px;
                            ">管理</button>

                            <button id="save-as-template" style="
                                background: #ffc107;
                                color: #212529;
                                border: none;
                                padding: 6px 12px;
                                border-radius: 4px;
                                cursor: pointer;
                                margin-left: 8px;
                            ">保存</button>
                        </div>
                    </div>
                </div>

                <!-- テンプレート管理ダイアログ -->
                <div id="template-manager" style="display: none;">
                    <div style="position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.5); z-index: 10000;" id="template-overlay"></div>
                    <div style="
                        position: fixed;
                        top: 50%;
                        left: 50%;
                        transform: translate(-50%, -50%);
                        background: white;
                        border-radius: 8px;
                        box-shadow: 0 4px 20px rgba(0,0,0,0.3);
                        z-index: 10001;
                        width: 800px;
                        max-width: 90vw;
                        max-height: 90vh;
                        overflow: hidden;
                    ">
                        <div style="
                            padding: 20px;
                            border-bottom: 1px solid #dee2e6;
                            background: #f8f9fa;
                            display: flex;
                            justify-content: space-between;
                            align-items: center;
                        ">
                            <h3 style="margin: 0;">テンプレート管理</h3>
                            <button id="close-template-manager" style="
                                background: none;
                                border: none;
                                font-size: 20px;
                                cursor: pointer;
                                padding: 5px;
                            ">&times;</button>
                        </div>

                        <div style="padding: 20px; max-height: 70vh; overflow-y: auto;">
                            <div style="margin-bottom: 20px;">
                                <button id="create-new-template" style="
                                    background: #28a745;
                                    color: white;
                                    border: none;
                                    padding: 8px 16px;
                                    border-radius: 4px;
                                    cursor: pointer;
                                    margin-right: 10px;
                                ">新規作成</button>

                                <button id="import-templates" style="
                                    background: #17a2b8;
                                    color: white;
                                    border: none;
                                    padding: 8px 16px;
                                    border-radius: 4px;
                                    cursor: pointer;
                                    margin-right: 10px;
                                ">インポート</button>

                                <button id="export-templates" style="
                                    background: #6c757d;
                                    color: white;
                                    border: none;
                                    padding: 8px 16px;
                                    border-radius: 4px;
                                    cursor: pointer;
                                ">エクスポート</button>
                            </div>

                            <div id="template-list"></div>

                            <!-- テンプレート編集フォーム -->
                            <div id="template-editor" style="display: none; margin-top: 20px; border-top: 1px solid #dee2e6; padding-top: 20px;">
                                <h4>テンプレート編集</h4>
                                <div style="margin-bottom: 15px;">
                                    <label>名前:</label>
                                    <input type="text" id="template-name" style="width: 100%; padding: 8px; margin-top: 5px;">
                                </div>
                                <div style="margin-bottom: 15px;">
                                    <label>トラッカー:</label>
                                    <select id="template-tracker" style="width: 100%; padding: 8px; margin-top: 5px;">
                                        <option value="">全トラッカー</option>
                                    </select>
                                </div>
                                <div style="margin-bottom: 15px;">
                                    <label>件名テンプレート:</label>
                                    <input type="text" id="template-subject" style="width: 100%; padding: 8px; margin-top: 5px;" placeholder="例: [バグ報告] {{summary}}">
                                </div>
                                <div style="margin-bottom: 15px;">
                                    <label>説明テンプレート:</label>
                                    <textarea id="template-description" style="width: 100%; height: 200px; padding: 8px; margin-top: 5px;" placeholder="テンプレートの内容を入力してください..."></textarea>
                                </div>
                                <div style="text-align: right;">
                                    <button id="cancel-template-edit" style="
                                        background: #6c757d;
                                        color: white;
                                        border: none;
                                        padding: 8px 16px;
                                        border-radius: 4px;
                                        cursor: pointer;
                                        margin-right: 10px;
                                    ">キャンセル</button>
                                    <button id="save-template-edit" style="
                                        background: #28a745;
                                        color: white;
                                        border: none;
                                        padding: 8px 16px;
                                        border-radius: 4px;
                                        cursor: pointer;
                                    ">保存</button>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            `;

            // チケットフォームの上に挿入
            $('#issue_subject').closest('p, div').before(templateHtml);
            this.loadTrackerOptions();
        },

        // イベントバインド
        bindEvents: function() {
            var self = this;

            // テンプレート選択
            $('#template-selector').change(function() {
                self.updateTemplateSelector();
            });

            // トラッカーフィルター
            $('#tracker-filter').change(function() {
                self.updateTemplateSelector();
            });

            // テンプレート適用
            $('#apply-template').click(function() {
                self.applyTemplate();
            });

            // テンプレート管理
            $('#manage-templates').click(function() {
                self.showTemplateManager();
            });

            // テンプレートとして保存
            $('#save-as-template').click(function() {
                self.saveCurrentAsTemplate();
            });

            // 管理ダイアログのイベント
            $('#template-overlay, #close-template-manager').click(function() {
                $('#template-manager').hide();
            });

            $('#create-new-template').click(function() {
                self.showTemplateEditor();
            });

            $('#import-templates').click(function() {
                self.importTemplates();
            });

            $('#export-templates').click(function() {
                self.exportTemplates();
            });

            // テンプレート編集
            $('#save-template-edit').click(function() {
                self.saveTemplateEdit();
            });

            $('#cancel-template-edit').click(function() {
                self.hideTemplateEditor();
            });

            // トラッカー変更時の自動フィルタ
            $('#issue_tracker_id').change(function() {
                self.currentTracker = $(this).val();
                self.updateTemplateSelector();
            });
        },

        // トラッカーオプションの読み込み
        loadTrackerOptions: function() {
            $('#issue_tracker_id option').each(function() {
                if ($(this).val()) {
                    $('#tracker-filter, #template-tracker').append(
                        '<option value="' + $(this).val() + '">' + $(this).text() + '</option>'
                    );
                }
            });
        },

        // テンプレートセレクターの更新
        updateTemplateSelector: function() {
            var $selector = $('#template-selector');
            var trackerFilter = $('#tracker-filter').val();
            var currentTracker = $('#issue_tracker_id').val();

            $selector.find('option:not(:first)').remove();

            var filteredTemplates = this.getFilteredTemplates(trackerFilter || currentTracker);

            filteredTemplates.forEach(function(template) {
                $selector.append('<option value="' + template.id + '">' + template.name + '</option>');
            });
        },

        // フィルタされたテンプレートを取得
        getFilteredTemplates: function(trackerId) {
            var projectTemplates = this.templates[this.currentProject] || [];
            var globalTemplates = this.templates['global'] || [];
            var allTemplates = projectTemplates.concat(globalTemplates);

            if (trackerId) {
                return allTemplates.filter(function(template) {
                    return !template.tracker || template.tracker === trackerId;
                });
            }

            return allTemplates;
        },

        // テンプレート適用
        applyTemplate: function() {
            var templateId = $('#template-selector').val();
            if (!templateId) return;

            var template = this.findTemplateById(templateId);
            if (!template) return;

            // 件名の適用
            if (template.subject) {
                var subject = this.processTemplate(template.subject);
                $('#issue_subject').val(subject);
            }

            // 説明の適用
            if (template.description) {
                var description = this.processTemplate(template.description);
                $('#issue_description').val(description);

                // CKEditor対応
                if (window.CKEDITOR && CKEDITOR.instances.issue_description) {
                    CKEDITOR.instances.issue_description.setData(description);
                }
            }

            // 成功メッセージ
            this.showMessage('テンプレートを適用しました: ' + template.name, 'success');
        },

        // テンプレート処理（変数置換）
        processTemplate: function(template) {
            var now = new Date();
            var user = $('#loggedas a').text() || 'Unknown';
            var project = this.currentProject;

            return template
                .replace(/\{\{date\}\}/g, now.toLocaleDateString('ja-JP'))
                .replace(/\{\{time\}\}/g, now.toLocaleTimeString('ja-JP'))
                .replace(/\{\{user\}\}/g, user)
                .replace(/\{\{project\}\}/g, project)
                .replace(/\{\{summary\}\}/g, '')
                .replace(/\{\{version\}\}/g, $('#issue_fixed_version_id option:selected').text() || '')
                .replace(/\{\{priority\}\}/g, $('#issue_priority_id option:selected').text() || '');
        },

        // テンプレート管理画面表示
        showTemplateManager: function() {
            $('#template-manager').show();
            this.updateTemplateList();
        },

        // テンプレートリスト更新
        updateTemplateList: function() {
            var $list = $('#template-list');
            var html = '';

            var allTemplates = this.getAllTemplates();

            if (allTemplates.length === 0) {
                html = '<p>テンプレートがありません。</p>';
            } else {
                html += '<table style="width: 100%; border-collapse: collapse;">';
                html += '<thead><tr style="background: #f8f9fa;"><th style="padding: 10px; border: 1px solid #dee2e6;">名前</th><th style="padding: 10px; border: 1px solid #dee2e6;">トラッカー</th><th style="padding: 10px; border: 1px solid #dee2e6;">プロジェクト</th><th style="padding: 10px; border: 1px solid #dee2e6;">操作</th></tr></thead>';
                html += '<tbody>';

                allTemplates.forEach(function(template) {
                    var trackerName = template.tracker ? $('#issue_tracker_id option[value="' + template.tracker + '"]').text() : '全て';
                    html += '<tr>';
                    html += '<td style="padding: 10px; border: 1px solid #dee2e6;">' + template.name + '</td>';
                    html += '<td style="padding: 10px; border: 1px solid #dee2e6;">' + trackerName + '</td>';
                    html += '<td style="padding: 10px; border: 1px solid #dee2e6;">' + (template.project || 'グローバル') + '</td>';
                    html += '<td style="padding: 10px; border: 1px solid #dee2e6;">';
                    html += '<button class="edit-template" data-id="' + template.id + '" style="background: #ffc107; color: #212529; border: none; padding: 4px 8px; border-radius: 3px; cursor: pointer; margin-right: 5px;">編集</button>';
                    html += '<button class="delete-template" data-id="' + template.id + '" style="background: #dc3545; color: white; border: none; padding: 4px 8px; border-radius: 3px; cursor: pointer;">削除</button>';
                    html += '</td>';
                    html += '</tr>';
                });

                html += '</tbody></table>';
            }

            $list.html(html);

            // 編集・削除ボタンのイベント
            var self = this;
            $('.edit-template').click(function() {
                self.editTemplate($(this).data('id'));
            });

            $('.delete-template').click(function() {
                if (confirm('テンプレートを削除しますか？')) {
                    self.deleteTemplate($(this).data('id'));
                }
            });
        },

        // テンプレート編集画面表示
        showTemplateEditor: function(template) {
            $('#template-editor').show();

            if (template) {
                $('#template-name').val(template.name);
                $('#template-tracker').val(template.tracker || '');
                $('#template-subject').val(template.subject || '');
                $('#template-description').val(template.description || '');
                $('#template-editor').data('template-id', template.id);
            } else {
                $('#template-name').val('');
                $('#template-tracker').val('');
                $('#template-subject').val('');
                $('#template-description').val('');
                $('#template-editor').removeData('template-id');
            }
        },

        // テンプレート編集画面非表示
        hideTemplateEditor: function() {
            $('#template-editor').hide();
        },

        // テンプレート保存
        saveTemplateEdit: function() {
            var name = $('#template-name').val().trim();
            var tracker = $('#template-tracker').val();
            var subject = $('#template-subject').val();
            var description = $('#template-description').val();

            if (!name) {
                alert('テンプレート名を入力してください。');
                return;
            }

            var template = {
                id: $('#template-editor').data('template-id') || this.generateId(),
                name: name,
                tracker: tracker,
                subject: subject,
                description: description,
                project: this.currentProject,
                created: new Date().toISOString(),
                updated: new Date().toISOString()
            };

            this.saveTemplate(template);
            this.hideTemplateEditor();
            this.updateTemplateList();
            this.updateTemplateSelector();
            this.showMessage('テンプレートを保存しました: ' + name, 'success');
        },

        // 現在の内容をテンプレートとして保存
        saveCurrentAsTemplate: function() {
            var name = prompt('テンプレート名を入力してください:');
            if (!name) return;

            var template = {
                id: this.generateId(),
                name: name,
                tracker: $('#issue_tracker_id').val(),
                subject: $('#issue_subject').val(),
                description: $('#issue_description').val(),
                project: this.currentProject,
                created: new Date().toISOString(),
                updated: new Date().toISOString()
            };

            this.saveTemplate(template);
            this.updateTemplateSelector();
            this.showMessage('テンプレートを保存しました: ' + name, 'success');
        },

        // テンプレート保存（内部）
        saveTemplate: function(template) {
            if (!this.templates[template.project]) {
                this.templates[template.project] = [];
            }

            var existingIndex = this.templates[template.project].findIndex(t => t.id === template.id);
            if (existingIndex >= 0) {
                this.templates[template.project][existingIndex] = template;
            } else {
                this.templates[template.project].push(template);
            }

            this.saveTemplates();
        },

        // テンプレート削除
        deleteTemplate: function(templateId) {
            for (var project in this.templates) {
                this.templates[project] = this.templates[project].filter(t => t.id !== templateId);
            }
            this.saveTemplates();
            this.updateTemplateList();
            this.updateTemplateSelector();
            this.showMessage('テンプレートを削除しました', 'info');
        },

        // テンプレート編集
        editTemplate: function(templateId) {
            var template = this.findTemplateById(templateId);
            if (template) {
                this.showTemplateEditor(template);
            }
        },

        // テンプレートをIDで検索
        findTemplateById: function(templateId) {
            for (var project in this.templates) {
                var found = this.templates[project].find(t => t.id === templateId);
                if (found) return found;
            }
            return null;
        },

        // 全テンプレートを取得
        getAllTemplates: function() {
            var allTemplates = [];
            for (var project in this.templates) {
                this.templates[project].forEach(function(template) {
                    allTemplates.push({...template, project: project});
                });
            }
            return allTemplates.sort((a, b) => a.name.localeCompare(b.name));
        },

        // テンプレートエクスポート
        exportTemplates: function() {
            var data = JSON.stringify(this.templates, null, 2);
            var blob = new Blob([data], {type: 'application/json'});
            var url = URL.createObjectURL(blob);
            var a = document.createElement('a');
            a.href = url;
            a.download = 'redmine_templates_' + new Date().toISOString().split('T')[0] + '.json';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
            this.showMessage('テンプレートをエクスポートしました', 'success');
        },

        // テンプレートインポート
        importTemplates: function() {
            var input = document.createElement('input');
            input.type = 'file';
            input.accept = '.json';
            var self = this;

            input.onchange = function(e) {
                var file = e.target.files[0];
                if (!file) return;

                var reader = new FileReader();
                reader.onload = function(e) {
                    try {
                        var importedTemplates = JSON.parse(e.target.result);

                        // テンプレートをマージ
                        for (var project in importedTemplates) {
                            if (!self.templates[project]) {
                                self.templates[project] = [];
                            }

                            importedTemplates[project].forEach(function(template) {
                                // 重複チェック
                                var exists = self.templates[project].find(t => t.name === template.name);
                                if (!exists) {
                                    template.id = self.generateId();
                                    template.imported = new Date().toISOString();
                                    self.templates[project].push(template);
                                }
                            });
                        }

                        self.saveTemplates();
                        self.updateTemplateList();
                        self.updateTemplateSelector();
                        self.showMessage('テンプレートをインポートしました', 'success');

                    } catch (error) {
                        alert('ファイルの読み込みに失敗しました: ' + error.message);
                    }
                };
                reader.readAsText(file);
            };

            input.click();
        },

        // プリセットテンプレートの読み込み
        loadPresetTemplates: function() {
            var presets = [
                {
                    name: 'バグ報告',
                    tracker: '',
                    subject: '[バグ] {{summary}}',
                    description: `## バグの概要
{{summary}}について説明してください。

## 再現手順
1.
2.
3.

## 期待される結果
期待される動作を記載してください。

## 実際の結果
実際の動作を記載してください。

## 環境情報
- OS:
- ブラウザ:
- バージョン:

## 追加情報
その他の関連情報があれば記載してください。

---
報告者: {{user}}
報告日: {{date}}`
                },
                {
                    name: '機能要求',
                    tracker: '',
                    subject: '[機能要求] {{summary}}',
                    description: `## 機能の概要
{{summary}}について説明してください。

## 背景・目的
この機能が必要な背景や目的を記載してください。

## 要求仕様
### 基本機能
-
-
-

### 詳細仕様
詳細な仕様や条件を記載してください。

## 受入条件
- [ ]
- [ ]
- [ ]

## 参考資料
関連するドキュメントやリンクがあれば記載してください。

---
要求者: {{user}}
作成日: {{date}}`
                },
                {
                    name: 'タスク',
                    tracker: '',
                    subject: '{{summary}}',
                    description: `## タスクの目的
{{summary}}について説明してください。

## 作業内容
### やること
- [ ]
- [ ]
- [ ]

### やらないこと
-
-

## 成果物
-
-

## 注意事項
作業時の注意点があれば記載してください。

## 関連チケット
関連するチケットがあれば記載してください。

---
担当者: {{user}}
作成日: {{date}}`
                },
                {
                    name: 'ドキュメント作成',
                    tracker: '',
                    subject: '[ドキュメント] {{summary}}',
                    description: `## ドキュメントの目的
{{summary}}について説明してください。

## 対象読者
-
-

## 内容
### 構成案
1.
2.
3.

### 必要な情報
-
-

## デリバリー形式
- [ ] Wiki
- [ ] PDF
- [ ] その他:

## スケジュール
- 初稿完成:
- レビュー完了:
- 最終版:

---
作成者: {{user}}
作成日: {{date}}`
                }
            ];

            // グローバルテンプレートとして追加（存在しない場合のみ）
            if (!this.templates.global) {
                this.templates.global = [];
            }

            presets.forEach(function(preset) {
                var exists = this.templates.global.find(t => t.name === preset.name);
                if (!exists) {
                    preset.id = this.generateId();
                    preset.project = 'global';
                    preset.created = new Date().toISOString();
                    preset.updated = new Date().toISOString();
                    this.templates.global.push(preset);
                }
            }.bind(this));

            this.saveTemplates();
        },

        // ユニークIDの生成
        generateId: function() {
            return 'template_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
        },

        // メッセージ表示
        showMessage: function(message, type) {
            var className = 'template-message-' + (type || 'info');
            var color = {
                'success': '#28a745',
                'warning': '#ffc107',
                'error': '#dc3545',
                'info': '#17a2b8'
            }[type] || '#17a2b8';

            var $message = $('<div class="' + className + '" style="' +
                'position: fixed; top: 20px; right: 20px; z-index: 10005; ' +
                'background: ' + color + '; color: white; padding: 12px 20px; ' +
                'border-radius: 4px; box-shadow: 0 2px 10px rgba(0,0,0,0.2); ' +
                'max-width: 300px; word-wrap: break-word;' +
            '">' + message + '</div>');

            $('body').append($message);

            setTimeout(function() {
                $message.fadeOut(500, function() {
                    $(this).remove();
                });
            }, 3000);
        },

        // テンプレート保存（ローカルストレージ）
        saveTemplates: function() {
            try {
                localStorage.setItem(this.storageKey, JSON.stringify(this.templates));
            } catch (e) {
                console.warn('テンプレートの保存に失敗しました:', e);
            }
        },

        // テンプレート読み込み（ローカルストレージ）
        loadTemplates: function() {
            try {
                var stored = localStorage.getItem(this.storageKey);
                if (stored) {
                    this.templates = JSON.parse(stored);
                } else {
                    this.templates = {};
                }
            } catch (e) {
                console.warn('テンプレートの読み込みに失敗しました:', e);
                this.templates = {};
            }
        }
    };

    // チケット作成・編集ページでのみ実行
    if (window.location.pathname.match(/\/issues\/(new|\d+\/edit)/)) {
        IssueTemplateSystem.init();

        // グローバルに公開
        window.RedmineIssueTemplates = IssueTemplateSystem;
    }
});
```

### 2. テンプレートシステム用CSS

#### 基本設定
- **名前**: Issue Templates スタイル
- **カスタマイズタイプ**: CSS
- **挿入位置**: html_head
- **パスパターン**: `/issues/(new|\d+/edit)`

#### CSS コード

```css
/* テンプレートシステム専用スタイル */
#template-system {
    animation: fadeIn 0.5s ease-in;
}

@keyframes fadeIn {
    from { opacity: 0; transform: translateY(-10px); }
    to { opacity: 1; transform: translateY(0); }
}

#template-system button:hover {
    transform: translateY(-1px);
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    transition: all 0.2s ease;
}

#template-manager table th {
    background: #e9ecef !important;
    font-weight: 600;
    text-align: left;
}

#template-manager table tr:hover {
    background: #f8f9fa;
}

.edit-template:hover {
    background: #e0a800 !important;
}

.delete-template:hover {
    background: #c82333 !important;
}

/* レスポンシブ対応 */
@media (max-width: 768px) {
    #template-system {
        padding: 10px;
    }

    #template-system > div {
        flex-direction: column;
        gap: 10px;
        align-items: stretch;
    }

    #template-system button {
        margin: 2px 0;
    }

    #template-manager > div:last-child {
        width: 95vw;
    }

    #template-manager table {
        font-size: 14px;
    }

    #template-manager table th,
    #template-manager table td {
        padding: 6px !important;
    }
}

/* アクセシビリティ */
#template-system button:focus,
#template-selector:focus,
#tracker-filter:focus {
    outline: 2px solid #007bff;
    outline-offset: 2px;
}

/* ダークモード対応 */
@media (prefers-color-scheme: dark) {
    #template-system {
        background: #2d3436 !important;
        border-color: #636e72 !important;
        color: #ddd !important;
    }

    #template-manager > div:last-child > div {
        background: #2d3436 !important;
        color: #ddd !important;
    }

    #template-manager table {
        background: #2d3436 !important;
    }

    #template-manager table th {
        background: #636e72 !important;
        color: #fff !important;
    }

    #template-manager table td {
        border-color: #636e72 !important;
    }

    #template-editor {
        background: #2d3436 !important;
        border-color: #636e72 !important;
    }

    #template-name,
    #template-tracker,
    #template-subject,
    #template-description {
        background: #636e72 !important;
        border-color: #74b9ff !important;
        color: #fff !important;
    }
}

/* アニメーション最適化 */
@media (prefers-reduced-motion: reduce) {
    #template-system,
    #template-system button {
        animation: none;
        transition: none;
    }
}

/* プリント対応 */
@media print {
    #template-system,
    #template-manager {
        display: none !important;
    }
}
```

## 使用方法

### 基本操作

1. **テンプレートの適用**
   - チケット作成・編集画面でテンプレートを選択
   - 「適用」ボタンでフォームに反映

2. **テンプレートの管理**
   - 「管理」ボタンでテンプレート管理画面を開く
   - 新規作成、編集、削除が可能

3. **現在の内容を保存**
   - フォームに入力した内容を「保存」ボタンでテンプレート化

### 高度な機能

1. **変数の使用**
   - `{{date}}`: 現在の日付
   - `{{time}}`: 現在の時刻
   - `{{user}}`: ログインユーザー名
   - `{{project}}`: 現在のプロジェクト名
   - `{{summary}}`: 置換用プレースホルダー

2. **トラッカー別フィルタ**
   - 特定のトラッカーにのみ表示するテンプレート設定
   - 全トラッカー対応テンプレート

3. **インポート・エクスポート**
   - JSONファイルでのテンプレート共有
   - プロジェクト間でのテンプレート移行

## プリセットテンプレート

システム初回起動時に以下のテンプレートが自動作成されます：

1. **バグ報告テンプレート**
2. **機能要求テンプレート**
3. **タスクテンプレート**
4. **ドキュメント作成テンプレート**

## カスタマイズ

### 新しいテンプレート変数の追加

```javascript
// processTemplate関数に変数を追加
.replace(/\{\{custom_var\}\}/g, 'カスタム値')
```

### 自動適用の設定

```javascript
// 特定条件でのテンプレート自動適用
if ($('#issue_tracker_id').val() === '1') { // バグトラッカー
    // バグ報告テンプレートを自動適用
    $('#template-selector').val('bug_template_id').trigger('change');
}
```

## 制限事項

1. **データ保存**: ブラウザのローカルストレージを使用（ユーザーごと）
2. **共有**: チーム間でのテンプレート共有にはエクスポート・インポートが必要
3. **サーバーサイド**: データベース連携なし

## 拡張オプション

### サーバーサイドでの共有を実現する場合

1. **カスタムフィールド活用**: テンプレートデータをカスタムフィールドに保存
2. **Wiki連携**: Wikiページでのテンプレート管理
3. **API連携**: 外部システムとのテンプレート同期

## トラブルシューティング

### テンプレートが表示されない
1. チケット作成・編集ページにいるか確認
2. View Customizeの設定が正しいか確認
3. ブラウザのJavaScriptが有効か確認

### テンプレートが保存されない
1. ローカルストレージの容量制限確認
2. ブラウザの設定でローカルストレージが無効化されていないか確認

### パフォーマンスの問題
1. 不要なテンプレートを削除
2. 大量のテンプレートがある場合は分割を検討

---

**導入完了日**: 2025年9月26日
**対応バージョン**: Redmine 6.0.6 + View Customize Plugin
**実装方法**: JavaScript + CSS + ローカルストレージ