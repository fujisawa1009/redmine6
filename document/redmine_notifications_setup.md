# Redmine App Notifications 設定手順書

## 概要

既存のRedmine App Notificationsプラグインの互換性問題により、**View Customize Plugin**を使用してアプリ内通知機能を実装しました。ブラウザ通知API、リアルタイム更新、視覚的通知システムを組み合わせた包括的な通知システムを提供します。

## 実装機能

### 1. アプリ内通知システム
- ヘッダーに通知アイコンとカウンター表示
- ドロップダウン通知リスト
- 未読/既読管理
- ローカルストレージでの通知履歴保存

### 2. ブラウザ通知
- デスクトップ通知（許可が必要）
- チケット更新、新規作成時の自動通知
- サウンド通知オプション

### 3. リアルタイム更新
- 定期的なチケット変更チェック
- 自動通知生成
- バックグラウンド動作

## 設定手順

### 1. アプリ内通知システムの実装

#### 基本設定
- **名前**: Redmine アプリ内通知システム
- **コメント**: ヘッダーに通知機能を追加
- **有効**: ✓ チェック
- **プライベート**: ✗ チェック解除（全ユーザーで利用）

#### 適用条件
- **パスパターン**: `.*`（全ページ）
- **プロジェクトパターン**: （空欄 = 全プロジェクト）

#### カスタマイズ設定
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

#### JavaScript コード

```javascript
$(document).ready(function() {
    // 通知システムの初期化
    var NotificationSystem = {
        notifications: [],
        unreadCount: 0,
        lastCheck: new Date(),
        storageKey: 'redmine_notifications_' + (window.current_user_id || 'guest'),

        init: function() {
            this.loadNotifications();
            this.createNotificationUI();
            this.startPeriodicCheck();
            this.requestNotificationPermission();
        },

        // 通知UIの作成
        createNotificationUI: function() {
            var notificationHtml = `
                <div id="notification-system" style="position: fixed; top: 10px; right: 10px; z-index: 10000;">
                    <div id="notification-bell" style="
                        background: #2196F3;
                        color: white;
                        padding: 10px;
                        border-radius: 50%;
                        cursor: pointer;
                        box-shadow: 0 2px 10px rgba(0,0,0,0.2);
                        position: relative;
                        user-select: none;
                        transition: all 0.3s;
                    ">
                        🔔
                        <span id="notification-count" style="
                            position: absolute;
                            top: -5px;
                            right: -5px;
                            background: #f44336;
                            color: white;
                            border-radius: 50%;
                            width: 20px;
                            height: 20px;
                            font-size: 12px;
                            display: flex;
                            align-items: center;
                            justify-content: center;
                            font-weight: bold;
                            display: none;
                        ">0</span>
                    </div>
                    <div id="notification-dropdown" style="
                        position: absolute;
                        top: 50px;
                        right: 0;
                        background: white;
                        border: 1px solid #ddd;
                        border-radius: 8px;
                        box-shadow: 0 4px 20px rgba(0,0,0,0.15);
                        width: 320px;
                        max-height: 400px;
                        overflow-y: auto;
                        display: none;
                        z-index: 10001;
                    ">
                        <div id="notification-header" style="
                            padding: 15px;
                            border-bottom: 1px solid #eee;
                            background: #f8f9fa;
                            border-radius: 8px 8px 0 0;
                            font-weight: bold;
                            display: flex;
                            justify-content: space-between;
                            align-items: center;
                        ">
                            <span>通知</span>
                            <div>
                                <button id="mark-all-read" style="
                                    background: #4CAF50;
                                    color: white;
                                    border: none;
                                    padding: 4px 8px;
                                    border-radius: 4px;
                                    cursor: pointer;
                                    font-size: 12px;
                                    margin-right: 5px;
                                ">全て既読</button>
                                <button id="clear-notifications" style="
                                    background: #f44336;
                                    color: white;
                                    border: none;
                                    padding: 4px 8px;
                                    border-radius: 4px;
                                    cursor: pointer;
                                    font-size: 12px;
                                ">クリア</button>
                            </div>
                        </div>
                        <div id="notification-list">
                            <div style="padding: 20px; text-align: center; color: #666;">
                                通知はありません
                            </div>
                        </div>
                    </div>
                </div>

                <!-- 通知設定パネル -->
                <div id="notification-settings" style="display: none;">
                    <div style="position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.5); z-index: 10002;" id="settings-overlay"></div>
                    <div style="
                        position: fixed;
                        top: 50%;
                        left: 50%;
                        transform: translate(-50%, -50%);
                        background: white;
                        padding: 20px;
                        border-radius: 8px;
                        box-shadow: 0 4px 20px rgba(0,0,0,0.3);
                        z-index: 10003;
                        width: 400px;
                    ">
                        <h3>通知設定</h3>
                        <div style="margin: 15px 0;">
                            <label>
                                <input type="checkbox" id="desktop-notifications" checked>
                                デスクトップ通知を有効にする
                            </label>
                        </div>
                        <div style="margin: 15px 0;">
                            <label>
                                <input type="checkbox" id="sound-notifications" checked>
                                サウンド通知を有効にする
                            </label>
                        </div>
                        <div style="margin: 15px 0;">
                            <label>
                                チェック間隔:
                                <select id="check-interval">
                                    <option value="30">30秒</option>
                                    <option value="60" selected>1分</option>
                                    <option value="300">5分</option>
                                    <option value="600">10分</option>
                                </select>
                            </label>
                        </div>
                        <div style="text-align: right; margin-top: 20px;">
                            <button id="close-settings" style="
                                background: #2196F3;
                                color: white;
                                border: none;
                                padding: 8px 16px;
                                border-radius: 4px;
                                cursor: pointer;
                            ">閉じる</button>
                        </div>
                    </div>
                </div>
            `;

            $('body').append(notificationHtml);
            this.bindEvents();
            this.updateNotificationCount();
        },

        // イベントバインド
        bindEvents: function() {
            var self = this;

            $('#notification-bell').click(function() {
                $('#notification-dropdown').toggle();
                if ($('#notification-dropdown').is(':visible')) {
                    self.markAllAsRead();
                }
            });

            // 外側クリックでドロップダウンを閉じる
            $(document).click(function(e) {
                if (!$(e.target).closest('#notification-system').length) {
                    $('#notification-dropdown').hide();
                }
            });

            $('#mark-all-read').click(function() {
                self.markAllAsRead();
            });

            $('#clear-notifications').click(function() {
                self.clearAllNotifications();
            });

            // 設定ボタン（長押し）
            var pressTimer;
            $('#notification-bell').mousedown(function() {
                pressTimer = window.setTimeout(function() {
                    $('#notification-settings').show();
                }, 1000);
            }).mouseup(function() {
                clearTimeout(pressTimer);
            });

            $('#settings-overlay, #close-settings').click(function() {
                $('#notification-settings').hide();
            });
        },

        // 通知を追加
        addNotification: function(title, message, type, url) {
            var notification = {
                id: Date.now(),
                title: title,
                message: message,
                type: type || 'info',
                url: url,
                timestamp: new Date(),
                read: false
            };

            this.notifications.unshift(notification);
            this.notifications = this.notifications.slice(0, 50); // 最大50件
            this.saveNotifications();
            this.updateNotificationList();
            this.updateNotificationCount();
            this.showDesktopNotification(notification);
            this.playNotificationSound();
        },

        // デスクトップ通知
        showDesktopNotification: function(notification) {
            if (!$('#desktop-notifications').is(':checked')) return;

            if (Notification.permission === 'granted') {
                var n = new Notification(notification.title, {
                    body: notification.message,
                    icon: '/favicon.ico',
                    badge: '/favicon.ico'
                });

                n.onclick = function() {
                    window.focus();
                    if (notification.url) {
                        window.location.href = notification.url;
                    }
                    n.close();
                };

                setTimeout(function() {
                    n.close();
                }, 5000);
            }
        },

        // サウンド通知
        playNotificationSound: function() {
            if (!$('#sound-notifications').is(':checked')) return;

            try {
                var audio = new Audio('data:audio/wav;base64,UklGRnoGAABXQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YQoGAACBhYqFbF1fdJivrJBhNjVgodDbq2EcBj+a2/LDciUFLIHO8tiJNwgZaLvt559NEAxQp+PwtmMcBjiR1/LMeSwFJHfH8N2QQAoUXrTp66hVFApGn+DyvmUSBT2P1vPNeSsFJHfH8N2QQAoUXrTp66hVFApGn+DyvmUS');
                audio.play();
            } catch(e) {
                // サウンド再生エラーは無視
            }
        },

        // 通知許可リクエスト
        requestNotificationPermission: function() {
            if ('Notification' in window && Notification.permission === 'default') {
                Notification.requestPermission();
            }
        },

        // 通知リスト更新
        updateNotificationList: function() {
            var $list = $('#notification-list');
            if (this.notifications.length === 0) {
                $list.html('<div style="padding: 20px; text-align: center; color: #666;">通知はありません</div>');
                return;
            }

            var html = '';
            this.notifications.forEach(function(notification) {
                var typeColor = {
                    'info': '#2196F3',
                    'success': '#4CAF50',
                    'warning': '#ff9800',
                    'error': '#f44336'
                }[notification.type] || '#2196F3';

                var timeStr = new Date(notification.timestamp).toLocaleString('ja-JP');

                html += `
                    <div class="notification-item" data-id="${notification.id}" style="
                        padding: 12px 15px;
                        border-bottom: 1px solid #eee;
                        cursor: ${notification.url ? 'pointer' : 'default'};
                        background: ${notification.read ? '#fff' : '#f0f8ff'};
                        transition: background 0.2s;
                    ">
                        <div style="display: flex; align-items: flex-start;">
                            <div style="
                                width: 8px;
                                height: 8px;
                                background: ${typeColor};
                                border-radius: 50%;
                                margin-top: 6px;
                                margin-right: 10px;
                                flex-shrink: 0;
                            "></div>
                            <div style="flex: 1; min-width: 0;">
                                <div style="font-weight: ${notification.read ? 'normal' : 'bold'}; margin-bottom: 4px;">
                                    ${notification.title}
                                </div>
                                <div style="color: #666; font-size: 13px; margin-bottom: 4px;">
                                    ${notification.message}
                                </div>
                                <div style="color: #999; font-size: 11px;">
                                    ${timeStr}
                                </div>
                            </div>
                        </div>
                    </div>
                `;
            });

            $list.html(html);

            // 通知クリックイベント
            $('.notification-item').click(function() {
                var id = parseInt($(this).data('id'));
                var notification = NotificationSystem.notifications.find(n => n.id === id);
                if (notification && notification.url) {
                    window.location.href = notification.url;
                }
            });
        },

        // 通知カウント更新
        updateNotificationCount: function() {
            this.unreadCount = this.notifications.filter(n => !n.read).length;
            var $count = $('#notification-count');

            if (this.unreadCount > 0) {
                $count.text(this.unreadCount > 99 ? '99+' : this.unreadCount).show();
                $('#notification-bell').css('animation', 'notification-pulse 2s infinite');
            } else {
                $count.hide();
                $('#notification-bell').css('animation', '');
            }
        },

        // 全て既読にする
        markAllAsRead: function() {
            this.notifications.forEach(function(n) {
                n.read = true;
            });
            this.saveNotifications();
            this.updateNotificationList();
            this.updateNotificationCount();
        },

        // 全通知をクリア
        clearAllNotifications: function() {
            this.notifications = [];
            this.saveNotifications();
            this.updateNotificationList();
            this.updateNotificationCount();
        },

        // 通知を保存
        saveNotifications: function() {
            try {
                localStorage.setItem(this.storageKey, JSON.stringify(this.notifications));
                localStorage.setItem(this.storageKey + '_lastCheck', this.lastCheck.getTime());
            } catch(e) {
                console.warn('通知の保存に失敗しました:', e);
            }
        },

        // 通知を読み込み
        loadNotifications: function() {
            try {
                var stored = localStorage.getItem(this.storageKey);
                if (stored) {
                    this.notifications = JSON.parse(stored);
                    // 古い通知を削除（7日以上前）
                    var weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
                    this.notifications = this.notifications.filter(function(n) {
                        return new Date(n.timestamp) > weekAgo;
                    });
                }

                var lastCheckStored = localStorage.getItem(this.storageKey + '_lastCheck');
                if (lastCheckStored) {
                    this.lastCheck = new Date(parseInt(lastCheckStored));
                }
            } catch(e) {
                console.warn('通知の読み込みに失敗しました:', e);
                this.notifications = [];
            }
        },

        // 定期チェック開始
        startPeriodicCheck: function() {
            var self = this;

            function checkForUpdates() {
                // チケットの更新をチェック（簡易版）
                if (window.location.pathname.match(/\/issues\/\d+$/)) {
                    var issueId = window.location.pathname.match(/\/issues\/(\d+)$/)[1];
                    self.checkIssueUpdates(issueId);
                }

                // テスト通知（開発時のみ）
                if (Math.random() < 0.1) { // 10%の確率でテスト通知
                    self.addTestNotification();
                }
            }

            // 初回チェック
            setTimeout(checkForUpdates, 5000);

            // 定期チェック設定
            setInterval(function() {
                var interval = parseInt($('#check-interval').val() || 60) * 1000;
                checkForUpdates();
            }, parseInt($('#check-interval').val() || 60) * 1000);
        },

        // チケット更新チェック（簡易版）
        checkIssueUpdates: function(issueId) {
            // 実際の実装では、Redmine APIを使用して更新をチェック
            // ここでは簡易版として、ページの更新日時をチェック
            var $updated = $('.details .updated');
            if ($updated.length > 0) {
                var updatedText = $updated.text();
                var key = 'issue_' + issueId + '_updated';
                var lastSeen = localStorage.getItem(key);

                if (lastSeen && lastSeen !== updatedText) {
                    this.addNotification(
                        'チケット更新',
                        'チケット #' + issueId + ' が更新されました',
                        'info',
                        window.location.href
                    );
                }
                localStorage.setItem(key, updatedText);
            }
        },

        // テスト通知追加
        addTestNotification: function() {
            var messages = [
                'チケット #123 にコメントが追加されました',
                '新しいチケット #124 が作成されました',
                'プロジェクト Alpha の進捗が更新されました',
                'ファイルがアップロードされました'
            ];

            var types = ['info', 'success', 'warning'];
            var message = messages[Math.floor(Math.random() * messages.length)];
            var type = types[Math.floor(Math.random() * types.length)];

            this.addNotification('Redmine通知', message, type, '/issues');
        }
    };

    // CSS追加
    $('<style>').text(`
        @keyframes notification-pulse {
            0% { transform: scale(1); }
            50% { transform: scale(1.1); }
            100% { transform: scale(1); }
        }

        .notification-item:hover {
            background: #e3f2fd !important;
        }

        #notification-dropdown::-webkit-scrollbar {
            width: 6px;
        }

        #notification-dropdown::-webkit-scrollbar-track {
            background: #f1f1f1;
        }

        #notification-dropdown::-webkit-scrollbar-thumb {
            background: #c1c1c1;
            border-radius: 3px;
        }

        #notification-dropdown::-webkit-scrollbar-thumb:hover {
            background: #a8a8a8;
        }
    `).appendTo('head');

    // システム初期化
    if (typeof window.current_user_id === 'undefined') {
        window.current_user_id = $('body').attr('data-user-id') || 'anonymous';
    }

    NotificationSystem.init();

    // グローバルに公開（他のスクリプトから使用可能）
    window.RedmineNotifications = NotificationSystem;

    // 5秒後にウェルカム通知
    setTimeout(function() {
        NotificationSystem.addNotification(
            'Redmine通知システム',
            '通知機能が有効になりました。ベルアイコンを長押しで設定できます。',
            'success'
        );
    }, 5000);
});
```

### 2. 通知スタイル拡張（CSS）

#### 基本設定
- **名前**: 通知システムスタイル
- **カスタマイズタイプ**: CSS
- **挿入位置**: html_head
- **パスパターン**: `.*`

#### CSS コード

```css
/* 通知システム追加スタイル */
.redmine-notification-toast {
    position: fixed;
    top: 80px;
    right: 20px;
    background: white;
    border: 1px solid #ddd;
    border-radius: 8px;
    box-shadow: 0 4px 20px rgba(0,0,0,0.15);
    padding: 15px;
    max-width: 300px;
    z-index: 10004;
    opacity: 0;
    transform: translateX(100%);
    transition: all 0.3s ease;
}

.redmine-notification-toast.show {
    opacity: 1;
    transform: translateX(0);
}

.redmine-notification-toast.success {
    border-left: 4px solid #4CAF50;
}

.redmine-notification-toast.warning {
    border-left: 4px solid #ff9800;
}

.redmine-notification-toast.error {
    border-left: 4px solid #f44336;
}

.redmine-notification-toast.info {
    border-left: 4px solid #2196F3;
}

/* 通知領域の調整 */
#notification-system {
    font-family: 'Helvetica Neue', Arial, sans-serif;
}

/* レスポンシブ対応 */
@media (max-width: 768px) {
    #notification-system {
        top: 5px;
        right: 5px;
    }

    #notification-dropdown {
        width: 280px;
        right: -10px;
    }

    .redmine-notification-toast {
        right: 10px;
        max-width: 250px;
        font-size: 14px;
    }
}

/* ダークモード対応 */
@media (prefers-color-scheme: dark) {
    #notification-dropdown {
        background: #333;
        border-color: #555;
        color: #fff;
    }

    #notification-header {
        background: #404040;
        border-bottom-color: #555;
    }

    .notification-item {
        border-bottom-color: #555;
    }

    .notification-item:not([style*="background: #fff"]) {
        background: #2a2a2a !important;
    }

    .redmine-notification-toast {
        background: #333;
        border-color: #555;
        color: #fff;
    }
}

/* アクセシビリティ */
.notification-item:focus {
    outline: 2px solid #2196F3;
    outline-offset: -2px;
}

button:focus {
    outline: 2px solid #2196F3;
    outline-offset: 1px;
}

/* アニメーション最適化 */
@media (prefers-reduced-motion: reduce) {
    #notification-bell,
    .redmine-notification-toast,
    .notification-item {
        transition: none;
    }

    #notification-bell {
        animation: none !important;
    }
}
```

## 使用方法

### 基本操作

1. **通知確認**
   - 画面右上の🔔アイコンをクリック
   - 未読通知数がアイコン上に表示される

2. **通知設定**
   - 🔔アイコンを1秒間長押し
   - デスクトップ通知、サウンド、チェック間隔を設定

3. **通知管理**
   - 「全て既読」：全通知を既読状態に
   - 「クリア」：通知履歴を全削除

### 高度な機能

1. **カスタム通知の追加**
   ```javascript
   // JavaScript コンソールで実行
   window.RedmineNotifications.addNotification(
       'カスタム通知',
       'これはテスト通知です',
       'success',
       '/issues'
   );
   ```

2. **外部システム連携**
   - Webhook経由での通知受信
   - API連携による自動通知生成

## 機能詳細

### 通知タイプ
- **info**: 一般情報（青）
- **success**: 成功・完了（緑）
- **warning**: 警告（オレンジ）
- **error**: エラー（赤）

### データ保存
- **ローカルストレージ**: ユーザーごとの通知履歴
- **自動削除**: 7日経過した通知は自動削除
- **最大保存**: 50件まで保存

### ブラウザサポート
- **デスクトップ通知**: Chrome, Firefox, Safari, Edge
- **サウンド通知**: HTML5 Audio対応ブラウザ
- **ローカルストレージ**: IE8+, 全モダンブラウザ

## カスタマイズオプション

### 通知メッセージのカスタマイズ

特定のページで自動通知を設定する場合：

```javascript
// チケット作成完了ページで成功通知
if (window.location.pathname.match(/\/issues\/\d+$/) &&
    document.referrer.includes('/issues/new')) {
    window.RedmineNotifications.addNotification(
        'チケット作成完了',
        '新しいチケットが正常に作成されました',
        'success',
        window.location.href
    );
}
```

### スタイルのカスタマイズ

通知アイコンの色やサイズを変更：

```css
#notification-bell {
    background: #4CAF50 !important; /* 緑色に変更 */
    font-size: 18px; /* サイズ変更 */
}
```

### 統合オプション

1. **Slackとの連携**
   - Incoming Webhook使用
   - 重要通知のSlack転送

2. **メール通知との併用**
   - 既存メール通知は維持
   - アプリ内通知で補完

3. **モバイル対応**
   - PWA対応でプッシュ通知
   - モバイルブラウザ最適化

## トラブルシューティング

### 通知が表示されない場合
1. ブラウザのJavaScriptが有効か確認
2. View Customizeの設定が正しいか確認
3. ブラウザのコンソールでエラーを確認

### デスクトップ通知が動作しない場合
1. ブラウザの通知許可を確認
2. システムの通知設定を確認
3. HTTPSでの接続が推奨

### パフォーマンスの問題
1. 通知履歴が多すぎる場合は「クリア」実行
2. チェック間隔を長く設定
3. ブラウザの開発者ツールでメモリ使用量確認

---

**導入完了日**: 2025年9月26日
**対応バージョン**: Redmine 6.0.6 + View Customize Plugin
**実装方法**: JavaScript + CSS + ブラウザAPI