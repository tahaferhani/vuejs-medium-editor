<template>
    <div>
        <!-- Editor Mode -->

        <div class="medium-editor-container" v-if="!readOnly">
            <insert-embed
                v-if="editor"
                :uploadUrl="options.uploadUrl"
                :uploadUrlHeader="options.uploadUrlHeader"
                :file_input_name="options.file_input_name"
                :file_size="options.file_size"
                :imgur_bool="options.imgur"
                :onChange="triggerChange"
                :editorRef="$refs.editor"
                :editor="editor"
                :hide-gist="hideGist"
                :hide-image="hideImage"
                v-on:uploaded="uploadedCallback"
            ></insert-embed>

            <list-handler v-if="editor" :editor="editor" :onChange="triggerChange"></list-handler>

            <span class="editor" v-bind:class="editorClass" v-html="content" ref="editor"></span>
        </div>

        <!-- Read Only Mode -->

        <read-mode v-if="readOnly" :content="modelValue"></read-mode>
    </div>
</template>

<script>
    import MediumEditor from "medium-editor";
    import InsertEmbed from "./libs/InsertEmbed";
    import ListHandler from "./libs/ListHandler";
    import ReadMode from "./libs/ReadMode";
    import _ from "underscore";
    import hljs from "highlight.js";

    export default {
        name: "medium-editor",
        props: ["modelValue", "options", "onChange", "prefill", "readOnly", "hideGist", "hideImage"],
        data() {
            return {
                editor: null,
                content: this.prefill,
                defaultOptions: {
                    forcePlainText: false,
                    placeholder: {
                        text: "Write something great!!"
                    },
                    uploadUrl: "https://api.imgur.com/3/image",
                    uploadUrlHeader: { Authorization: "Client-ID db856b43cc7f441" },
                    file_input_name: "image",
                    file_size: 1024 * 1024 * 10,
                    imgur: true,
                    toolbar: {
                        buttons: ["bold", "italic", "quote", "h1", "h2", "h3", "h4", "h5", "anchor"]
                    }
                },
                hasContent: false,
                autoLink: true
            };
        },
        watch: {
            modelValue(content) {
                if (this.$refs.editor != document.activeElement) {
                    this.content = content;
                    this.$nextTick(() => {
                        this.$refs.editor.focus();
                        this.hasContent = /<[a-z][\s\S]*>/i.test(content) && content != "<p><br></p>";
                    });
                }
            }
        },
        computed: {
            editorOptions() {
                return _.extend(this.defaultOptions, this.options);
            },
            editorClass() {
                return {
                    "has-content": this.hasContent
                };
            }
        },
        components: {
            InsertEmbed,
            ListHandler,
            ReadMode
        },
        mounted() {
            this.addClassToPre();
            if (!this.readOnly) {
                this.createElm();
            }
        },
        methods: {
            createElm() {
                this.editor = new MediumEditor(this.$refs.editor, this.editorOptions);
                if (this.prefill) {
                    this.hasContent = /<[a-z][\s\S]*>/i.test(this.prefill) && this.prefill != "<p><br></p>";
                    this.$refs.editor.focus();
                }
                this.editor.subscribe("editableInput", this.triggerChange);
            },
            destroyElm() {
                this.editor.destroy();
            },
            triggerChange() {
                this.addClassToPre();
                const content = this.editor.getContent();
                setTimeout(() => {
                    this.hasContent = /<[a-z][\s\S]*>/i.test(content) && content != "<p><br></p>";
                }, 0);
                this.$emit("update:modelValue", content);
                if (this.onChange) {
                    this.onChange(content);
                }
            },
            uploadedCallback(url) {
                // console.log("callback")
                this.$emit("uploaded", url);
            },
            addClassToPre() {
                hljs.configure({});
                const brPlugin = {
                    "before:highlightBlock": ({ block }) => {
                        block.innerHTML = block.innerHTML.replace(/<br[ /]*>/g, "\n");
                    },
                    "after:highlightBlock": ({ result }) => {
                        result.value = result.value.replace(/\n/g, "<br>");
                    }
                };

                hljs.addPlugin(brPlugin);
                document.querySelectorAll("pre").forEach(block => {
                    hljs.highlightElement(block);
                    block.setAttribute("spellcheck", "false");
                });
            }
        },
        unmounted() {
            this.destroyElm();
        }
    };
</script>
