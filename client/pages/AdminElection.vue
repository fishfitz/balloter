<template>
    <div>
        <page-title :title="title"></page-title>

        <load-or-error :error="error" :loading="loading"></load-or-error>

        <div v-if="name || creation" class="form">

            <router-link v-if="slug" :to="'/election/' + slug">
                <span> ⇒ </span> Aller au vote
            </router-link>

            <div v-if="!creation">
                <b> Statut : </b>&nbsp;
                <election-state :ongoing="ongoing" :broadcasted="broadcasted"></election-state>
            </div>

            <div v-if="broadcasted" class='error'> ℹ Vous ne pouvez pas éditer une élection qui a été diffusée. </div>

            <div v-else>
                <label>
                    Nom de l'élection
                    <input type="text" v-model="name" @input="changed = true"/>
                </label>
                <label>
                    Description <em> (markdown autorisé) </em>
                    <textarea v-model="description" @input="changed = true"></textarea>
                </label>

                <h3> Choix </h3>
                <div class="choice" v-for="(choice, index) in choices">
                    <h4>
                        {{ index + 1 }}.
                        <input type="button" @click="deleteChoice(index)" value="✘ Supprimer ce choix"/>
                    </h4>
                    <label>
                        Nom du choix
                        <input type="text" v-model="choice.name" @input="changed = true"/>
                    </label>
                    <label>
                        Description <em> (markdown autorisé) </em>
                        <textarea v-model="choice.description" @input="changed = true"></textarea>
                    </label>
                </div>
                <input type="button" @click="addChoice" value="✚ Ajouter un choix"/>
            </div>

            <div class="actions">
                <async-button v-if="!broadcasted"
                    :disabled="!name || !this.changed" value="✓ Sauvegarder"
                    :request="saveOrUpdate()" @done="writeUp">
                </async-button>
                <async-button v-if="canBroadcast"
                    value="🔗 Diffuser sur discord"
                    :request="broadcast()" @done="broadcasted = true">
                </async-button>
                <async-button v-if="slug && ongoing && broadcasted"
                    value="🔒 Clore le vote"
                    :request="close()" @done="ongoing = false">
                </async-button>
                <async-button v-if="slug"
                    value="✘ Supprimer l'élection"
                    :request="deleteElection()" @done="$router.push('/')">
                </async-button>
            </div>
        </div>
    </div>
</template>

<script>
    module.exports = {
        props: ['creation'],
        components: {
            PageTitle: require('../components/PageTitle.vue'),
            AsyncButton: require('../components/AsyncButton.vue'),
            LoadOrError: require('../components/LoadOrError.vue'),
            ElectionState: require('../components/ElectionState.vue')
        },
        mixins: [require('../mixins/election.js')],
        computed: {
            title() {
                if (this.slug) {
                    return 'Editer une élection';
                }
                return 'Créer une élection';
            },
            actualChoices() {
                return this.choices.filter((c) => {
                    return c.name.slice();
                });
            },
            canBroadcast() {
                return this.slug && this.ongoing && this.actualChoices.length > 1 && this.name;
            }
        },
        methods: {
            afterWrite(body) {
                if (this.slug !== body.slug) {
                    this.$router.push('/edit/' + body.slug);
                }
                else {
                    this.changed = false;
                }
            },
            saveOrUpdate() {
                return () => {
                    if (this.slug) {
                        return this.update();
                    }
                    return this.save();
                };
            },
            save() {
                return this.$http.post('/api/election/', {
                    name: this.name,
                    description: this.description,
                    choices: this.choices.filter((c) => {
                        return c.name.slice();
                    })
                });
            },
            update() {
                return this.$http.put('/api/election/' + this.slug, {
                    name: this.name,
                    description: this.description,
                    choices: this.actualChoices
                });
            },
            broadcast() {
                return () => this.$http.post('/api/election/' + this.slug, {
                    role: 'MEMBRE'
                });
            },
            close() {
                return () => this.$http.patch('/api/election/' + this.slug, {
                    ongoing: false
                });
            },
            deleteElection() {
                return () => this.$http.delete('/api/election/' + this.slug);
            },
            addChoice() {
                this.changed = true;
                this.choices.push({
                    name: '',
                    description: ''
                });
            },
            deleteChoice(index) {
                this.changed = true;
                this.choices.splice(index, 1);
            }
        }
    };
</script>

<style lang="less" scoped>
    @import "../styles/colors.less";

    div.form div:first-child, div.form div:nth-child(2) {
        padding-bottom: 1.8em;
    }

    a {
        display: inline-block;
        font-size: 1em;
        padding-bottom: 1.5em;
        font-weight: bold;

        span {
            color: @header-color;
        }

        &:hover {
            text-decoration: underline;
        }
    }

    div.choice {
        border-bottom: 1px dashed rgba(0, 0, 0, 0.2);
        margin-bottom: 1em;
    }

    div.actions {
        margin-top: 2em;
        margin-bottom: 2em;
    }
</style>