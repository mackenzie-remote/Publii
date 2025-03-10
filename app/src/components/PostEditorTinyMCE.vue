<template>
    <div class="post-editor post-editor-wysiwyg" ref="post-editor">
        <topbar-appbar />
        <post-editor-top-bar />

        <div class="post-editor-wrapper">
            <div :class="{
                'post-editor-form': true,
                'writers-panel-open': writersPanelOpen,
                'sidebar-open': sidebarVisible
            }">
                <div>
                    <div
                        id="post-title"
                        ref="post-title"
                        class="post-editor-form-title"
                        contenteditable="true"
                        :data-translation="$t('post.addPostTitle')"
                        :spellcheck="$store.state.currentSite.config.spellchecking"
                        @paste.prevent="pasteTitle"
                        @keydown="detectEnterInTitle"
                        @input="updateTitle" />

                    <editor ref="tinymceEditor" />

                    <input
                        name="image"
                        id="post-editor-fake-image-uploader"
                        class="is-hidden"
                        type="file" />
                </div>
            </div>

            <writers-panel :isVisible="writersPanelOpen" />

            <p-button
                id="post-stats-button"
                type="clean-invert icon small"
                icon="stats"
                :title="$t('editor.togglePostStatsPanel')"
                @click.native="togglePostStats">
                <template v-if="!writersPanelOpen">{{ $t('editor.viewStats') }}</template>
                <template v-else>{{ $t('editor.hideStats') }}</template>
            </p-button>

            <sidebar :isVisible="sidebarVisible" />
            <author-popup />
            <date-popup />
            <source-code-editor ref="source-code-editor" />
            <link-popup />
            <link-toolbar />
            <gallery-popup />
            <inline-editor ref="inline-editor" />
        </div>
    </div>
</template>

<script>
import Vue from 'vue';
import PostEditorSidebar from './post-editor/Sidebar';
import InlineEditor from './post-editor/InlineEditor';
import PostEditorSourceCode from './post-editor/SourceCodeEditor';
import PostEditorWritersPanel from './post-editor/WritersPanel';
import AuthorPopup from './post-editor/AuthorPopup';
import DatePopup from './post-editor/DatePopup';
import LinkPopup from './post-editor/LinkPopup';
import LinkToolbar from './post-editor/LinkToolbar';
import GalleryPopup from './post-editor/GalleryPopup';
import Editor from './post-editor/Editor';
import TopBarAppBar from './TopBarAppBar';
import PostEditorTopBar from './post-editor/TopBar';
import PostHelper from './post-editor/PostHelper';
import Utils from './../helpers/utils';
import PostEditorsCommon from './mixins/PostEditorsCommon';

export default {
    name: 'post-editor-tinymce',
    editorType: 'tinymce',
    mixins: [
        PostEditorsCommon
    ],
    components: {
        'author-popup': AuthorPopup,
        'date-popup': DatePopup,
        'link-popup': LinkPopup,
        'gallery-popup': GalleryPopup,
        'link-toolbar': LinkToolbar,
        'editor': Editor,
        'inline-editor': InlineEditor,
        'sidebar': PostEditorSidebar,
        'source-code-editor': PostEditorSourceCode,
        'topbar-appbar': TopBarAppBar,
        'writers-panel': PostEditorWritersPanel,
        'post-editor-top-bar': PostEditorTopBar
    },
    data () {
        return {
            postID: this.$route.params.post_id || 0,
            newPost: true,
            writersPanelOpen: localStorage.getItem('publii-writers-panel') === 'opened',
            postSlugEdited: false,
            possibleDataLoss: false,
            unwatchDataLoss: null,
            sidebarVisible: false,
            postData: {
                editor: this.$options.editorType,
                title: '',
                text: '',
                slug: '',
                author: 1,
                tags: [],
                mainTag: '',
                template: '',
                creationDate: {
                    text: '',
                    timestamp: 0
                },
                modificationDate: {
                    text: '',
                    timestamp: 0
                },
                isFeatured: false,
                isHidden: false,
                isExcludedOnHomepage: false,
                status: '',
                metaTitle: '',
                metaDescription: '',
                metaRobots: 'index, follow',
                canonicalUrl: '',
                featuredImage: {
                    path: '',
                    alt: '',
                    caption: '',
                    credits: ''
                },
                postViewOptions: {}
            }
        };
    },
    computed: {
        isEdit () {
            return !!this.postID;
        },
        extensionsPath () {
            return [
                'file:///',
                this.$store.state.currentSite.siteDir,
                '/input/themes/',
                this.$store.state.currentSite.config.theme,
                '/'
            ].join('');
        }
    },
    mounted () {
        window.onbeforeunload = e => {
            if (this.possibleDataLoss) {
                e.returnValue = false;

                this.$bus.$emit('confirm-display', {
                    message: this.$t('core.sureYouWantQuit'),
                    isDanger: true,
                    okClick: () => {
                        window.onbeforeunload = null;
                        mainProcessAPI.send('app-close', true);
                    }
                });
            }
        };

        if (this.isEdit) {
            this.newPost = false;
            this.loadPostData();
        } else {
            this.$refs['tinymceEditor'].init();
            this.setDataLossWatcher();
        }

        this.$bus.$on('date-changed', (timestamp) => {
            let format = 'MMM DD, YYYY  HH:mm';

            if(this.$store.state.app.config.timeFormat == 12) {
                format = 'MMM DD, YYYY  hh:mm a';
            }

            this.postData.creationDate.timestamp = timestamp;
            this.postData.creationDate.text = this.$moment(this.postData.creationDate.timestamp).format(format);
        });

        this.$bus.$on('post-editor-possible-data-loss', () => {
            this.possibleDataLoss = true;
        });

        this.$refs['post-title'].focus();
    },
    methods: {
        updateTitle () {
            this.postData.title = this.$refs['post-title'].innerText.replace(/\n/gmi, ' ');
            this.updateSlug();
        },
        detectEnterInTitle (event) {
            if (event.code === 'Enter' && !event.isComposing) {
                event.preventDefault();
                this.$refs['tinymceEditor'].focus();
            }
        },
        loadPostData () {
            // Send request for a post to the back-end
            mainProcessAPI.send('app-post-load', {
                'site': this.$store.state.currentSite.config.name,
                'id': this.postID
            });

            // Load post data
            mainProcessAPI.receiveOnce('app-post-loaded', (data) => {
                if (data !== false && this.postID !== 0) {
                    let loadedPostData = PostHelper.loadPostData(data, this.$store, this.$moment);
                    this.postData = Utils.deepMerge(this.postData, loadedPostData);
                    this.$refs['post-title'].innerText = this.postData.title;
                    $('#post-editor').val(this.postData.text);
                    this.$refs['tinymceEditor'].init();
                }

                // Add custom editor script
                if(
                    this.$store.state.currentSite.themeSettings &&
                    this.$store.state.currentSite.themeSettings.extensions &&
                    this.$store.state.currentSite.themeSettings.extensions.postEditorCustomScript
                ) {
                    let customEditorScriptPath = this.extensionsPath + 'tinymce.script.js';

                    if (!document.querySelector('custom-post-editor-script')) {
                        $(document.body).append(
                            // It seems that Webpack goes crazy when it sees 'script' tag :)
                            $('<' + 'script' + ' id="custom-post-editor-script" src="' + customEditorScriptPath + '"></' + 'script' + '>')
                        );
                    }
                }

                setTimeout(() => {
                    this.possibleDataLoss = false;
                    this.setDataLossWatcher();
                }, 100);
            });
        },
        setDataLossWatcher () {
            this.unwatchDataLoss = this.$watch('postData', (newValue, oldValue) => {
                this.possibleDataLoss = true;
                this.unwatchDataLoss();
            }, { deep: true });
        },
        async savePost (newPostStatus, preview = false, closeEditor = false) {
            if (this.postData.title.trim() === '') {
                this.$bus.$emit('alert-display', {
                    message: this.$t('editor.cantSavePostWithEmptyTitle')
                });

                return;
            }

            tinymce.triggerSave();
            let postData = await PostHelper.preparePostData(newPostStatus, this.postID, this.$store, this.postData);

            if(!preview) {
                this.savingPost(newPostStatus, postData, closeEditor);
            } else {
                this.$bus.$emit('rendering-popup-display', {
                    itemID: this.postID,
                    postData: postData,
                    postOnly: true
                });
            }
        },
        savingPost (newStatus, postData, closeEditor = false) {
            // Send form data to the back-end
            mainProcessAPI.send('app-post-save', postData);

            // Post save
            mainProcessAPI.receiveOnce('app-post-saved', (data) => {
                if (this.postID === 0) {
                    this.postID = data.postID;
                }

                if (data.posts) {
                    this.savedPost(newStatus, data, closeEditor);
                } else {
                    alert(this.$t('editor.errorOccurred'));
                }
            });
        },
        savedPost (newStatus, updatedData, closeEditor = false) {
            this.$store.commit('refreshAfterPostUpdate', updatedData);

            if (closeEditor) {
                this.closeEditor();
                return;
            } else {
                this.$refs['tinymceEditor'].editorInstance.updatePostID(this.postID);
            }

            this.$router.push('/site/' + this.$route.params.name + '/posts/editor/tinymce/' + this.postID);
            let message = this.$t('editor.changesSaved');

            if (this.newPost) {
                this.newPost = false;

                if (newStatus === 'draft') {
                    message = this.$t('editor.newDraftCreated');
                } else {
                    message = this.$t('editor.newPostCreated');
                }
            }

            this.$bus.$emit('message-display', {
                message: message,
                type: 'success',
                lifeTime: 3
            });

            this.loadPostData();
            this.possibleDataLoss = false;
        },
        togglePostStats () {
            if (this.writersPanelOpen) {
                this.writersPanelOpen = false;
                localStorage.setItem('publii-writers-panel', 'closed');
            } else {
                this.writersPanelOpen = true;
                localStorage.setItem('publii-writers-panel', 'opened');
            }
        },
        pasteTitle (e) {
            let text = (e.originalEvent || e).clipboardData.getData('text/plain').replace(/\n/gmi, '');
            document.execCommand('insertText', false, text);
        }
    },
    beforeDestroy () {
        if (this.unwatchDataLoss) {
            this.unwatchDataLoss();
        }

        $('#custom-post-editor-script').remove();
        this.$bus.$off('date-changed');
        this.$bus.$off('post-editor-possible-data-loss');
        window.onbeforeunload = null;
    }
};
</script>

<style lang="scss">
@import '../scss/variables.scss';
@import '../scss/mixins.scss';
@import '../scss/editor/post-editors-common.scss';
@import '../scss/editor/editor-overrides.scss';

.post-editor {
    overflow-x: hidden;
    position: relative;
    width: 100%;
    z-index: 2;

    &-form {
        #post-title {
            border: none;
            box-shadow: none;
            color: var(--headings-color);
            display: block;
            font-family: -apple-system, BlinkMacSystemFont, Arial, "Segoe UI", "Roboto", "Oxygen", "Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue", sans-serif;
            font-size: 3.6rem;
            font-weight: var(--font-weight-bold);
            letter-spacing: var(--letter-spacing);
            line-height: 1.2;
            margin: 0 10% 2.6rem;
            padding: 0;
            text-align: center;
            width: 80%;

            &:empty {
                color: var(--gray-4);

                &:before {
                    content: attr(data-translation);
                }

                &:focus:before {
                    content: "";
                }
            }
        }

        #post-editor_ifr {
            height: calc( 100vh - 30rem )!important;
        }
    }
}

@media (min-width: 1800px) {
    .post-editor-form #post-title {
        margin: 0 auto 2.6rem;
        max-width: calc(100vw - 880px);
        width: 100%;
    }
}

/*
 * Special styles for win & linux
 */

body[data-os="win"] {
    .post-editor-form {
        #post-editor_ifr {
            height: calc( 100vh - 31rem )!important;
        }
    }

    .post-editor {
        #inline-toolbar {
            padding-top: 0;
            padding-bottom: 0;

            select {
                padding-top: 0;
                padding-bottom: 0;
                top: 0;
            }
        }

        #link-toolbar {
            padding-top: .75rem;
        }
    }
}

body[data-os="linux"] {
    .post-editor-form {
        #post-editor_ifr {
            height: calc( 100vh - 26.8rem )!important;
        }
    }
}
</style>
