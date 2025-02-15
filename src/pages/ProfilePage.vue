<script>
import axios from 'axios';
 // Import Swiper Vue.js components
 import { Swiper, SwiperSlide } from 'swiper/vue';

// Import Swiper styles
import 'swiper/css';

import 'swiper/css/effect-coverflow';
import 'swiper/css/pagination';


// import required modules
import { EffectCoverflow, Pagination } from 'swiper/modules';
export default {
    data() {
        return {
            apiUrl: 'http://127.0.0.1:8000/api/',
            singleMusician: null,
            demoPath: '',
            contactForm: {
                firstname: '',
                lastname: '',
                email: '',
                message: '',
                user_id: ''
            },
                reviewForm: {
                firstname: '',
                lastname: '',
                description: '',
                user_id: ''
            },
                voteForm: {
                user_id: '',
                vote: null
            },
            isSubmitting: false,
            reviewErrorMessage: '', // Nuova variabile per i messaggi di errore del form di recensione
            contactErrorMessage: '', // Nuova variabile per i messaggi di errore del form di contatto
            showReviewFormError: false, // Variabile per mostrare gli errori nel form di recensione
            showContactFormError: false, // Variabile per mostrare gli errori nel form di contatto
            ratings: [], // Dichiarazione della proprietà ratings
            successMessage: '',
            showError: '',
            activeStar: null,
            averageVote: 0,
            isHovered: false,
            numericVote: null, // Aggiungi questa variabile per memorizzare il voto numerico
            selectedStars: 0, // Aggiungi questa proprietà per tracciare il numero di stelle selezionate
            showReviewForm: false,
            showOffcanvas: false,
            reviews: [], // Tutta la lista delle review
            initialSlideIndex: 2
            
        };
    },
    components: {
      Swiper,
      SwiperSlide,
    },
    setup() {
      return {
        modules: [EffectCoverflow, Pagination],
      };
    },
    computed: {
        visibleReviews() {
        // Return only the first 3 reviews to display on the page
        return this.singleMusician.reviews.slice(0, 3);
        },
        remainingReviews() {
        // Return all reviews after the first 3
        return this.singleMusician.reviews.slice(3);
        }
    },
    methods: {
        async fetchUserData() {
        try {
            const response = await axios.get(`${this.apiUrl}users/${this.$route.params.name}`);
            this.singleMusician = response.data.result;
            this.demoPath = `http://127.0.0.1:8000/storage/${this.singleMusician.user_details.demo}`;
            this.contactForm.user_id = this.singleMusician.id;
            this.reviewForm.user_id = this.singleMusician.id;
            this.voteForm.user_id = this.singleMusician.id;
            this.calculateRoundedAverageVote(); // Calcola la media dei voti quando si caricano i dati dell'utente
            console.log(this.singleMusician);
        } catch (error) {
            this.errorMessage = 'Errore durante la chiamata API';
            console.error('Errore durante la chiamata API:', error);
        }
        },
        async sendMessage() {
            try {
                this.contactErrorMessage = ''; // Resetta eventuali messaggi di errore precedenti
                this.successMessage = ''; // Resetta eventuali messaggi di successo precedenti
                this.showError = true; // Mostra gli errori dopo il tentativo di invio

                // Validazione dei campi
                if (!this.contactForm.firstname.trim()) {
                    throw new Error('Inserisci il nome.');
                }
                if (!this.contactForm.lastname.trim()) {
                    throw new Error('Inserisci il cognome.');
                }
                if (!this.contactForm.email.trim()) {
                    throw new Error('Inserisci l\'email.');
                }
                // Validazione dell'email utilizzando una espressione regolare (regex)
                const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
                if (!emailRegex.test(this.contactForm.email.trim())) {
                    throw new Error('Email non valida.');
                }
                if (!this.contactForm.message.trim()) {
                    throw new Error('Inserisci il messaggio.');
                }
                if (this.contactForm.message.trim().length < 5) {
                    throw new Error('Il messaggio deve essere lungo almeno 5 caratteri.');
                }

                // Invia il messaggio
                await axios.post(`${this.apiUrl}messages`, this.contactForm);
                this.successMessage = 'Messaggio inviato con successo';
                this.resetForm();
            } catch (error) {
                // Gestisci gli errori del form di contatto
                this.contactErrorMessage = error.message;
                console.error('Errore durante l\'invio del messaggio:', error);
            } finally {
                // Rimuove i messaggi dopo 1 secondo
                setTimeout(() => {
                    this.contactErrorMessage = ''; // Correzione della variabile
                    this.successMessage = '';
                }, 2000);
            }
        },
        async submitReview() {
            try {
                this.reviewErrorMessage = ''; // Resetta eventuali messaggi di errore precedenti
                this.successMessage = ''; // Resetta eventuali messaggi di successo precedenti

                // Invia la recensione con il voto solo se numericVote non è null
                if (this.numericVote !== null) {
                this.reviewForm.vote = this.numericVote; // Converte il voto numerico in una stringa
                }

                // Verifica se si sta inviando solo il voto
                const isVoteOnly = this.voteForm.vote && !this.reviewForm.description && !this.reviewForm.firstname && !this.reviewForm.lastname;

                // Validazione dei campi se si sta inviando una recensione
                if (!isVoteOnly && this.reviewForm.description.trim()) {
                if (!this.reviewForm.firstname.trim()) {
                    throw new Error('Inserisci il nome.');
                }
                if (!this.reviewForm.lastname.trim()) {
                    throw new Error('Inserisci il cognome.');
                }
                if (!this.reviewForm.description.trim()) {
                    throw new Error('Inserisci la recensione.');
                }
                }

                // Invia la recensione solo se non si sta inviando solo il voto
                if (!isVoteOnly) {
                // Invia la recensione
                this.reviewForm.user_id = this.singleMusician.id;
                await axios.post(`${this.apiUrl}reviews`, this.reviewForm);
                this.successMessage = 'Recensione inviata con successo';
                this.resetReviewForm();
                }

                // Invia il voto, se presente
                if (this.voteForm.vote) {
                await this.submitVote();
                }
            } catch (error) {
                // Gestisci gli errori del form di recensione
                if (error.message === 'Inserisci il nome.' || error.message === 'Inserisci il cognome.' || error.message === 'Inserisci la recensione.') {
                this.reviewErrorMessage = error.message;
                } else {
                this.reviewErrorMessage = 'Errore durante l\'invio della recensione, inserisci i campi mancanti';
                }
                console.error('Errore durante l\'invio della recensione:', error);
            } finally {
                // Rimuove i messaggi dopo 1 secondo
                setTimeout(() => {
                this.reviewErrorMessage = '';
                this.successMessage = '';
                }, 2000);
            }
        },
        async submitVote() {
            try {
                this.contactErrorMessage = ''; // Resetta eventuali messaggi di errore precedenti
                this.successMessage = ''; // Resetta eventuali messaggi di successo precedenti

                if (!this.voteForm.vote) {
                    throw new Error('Seleziona un voto prima di inviare.');
                }

                // Imposta l'id dell'utente nel voto
                this.voteForm.user_id = this.singleMusician.id;
                const response = await axios.post(`${this.apiUrl}votes`, this.voteForm);
                console.log('Voto inviato con successo:', response.data);

                // Aggiungi un messaggio di successo per il voto
                this.successMessage += 'Voto inviato con successo. ';
            } catch (error) {
                this.contactErrorMessage = error.message || 'Errore durante l\'invio del voto.';
                console.error('Errore durante l\'invio del voto:', error);
            } finally {
                // Reimposta il form del voto
                this.voteForm.vote = null;
            }
        },
        // Metodo per ripristinare il form della recensione
        resetReviewForm() {
            this.reviewForm = {
                firstname: '',
                lastname: '',
                description: '',
                user_id: this.singleMusician.id
            };
        },
        // Metodo per calcolare la media dei voti
        calculateRoundedAverageVote() {
        if (this.singleMusician && this.singleMusician.votes && this.singleMusician.votes.length > 0) {
            let totalVotes = 0;

            for (let i = 0; i < this.singleMusician.votes.length; i++) {
            const elem = parseInt(this.singleMusician.votes[i].vote);
            if (!isNaN(elem)) {
                totalVotes += elem;
            }
            }

            const average = totalVotes / this.singleMusician.votes.length;
            console.log(average)
            // Arrotonda la media al 0.5 più vicino
            this.averageVote = Math.round(average * 2) / 2;

            console.log('Media voti:', this.averageVote);
        } else {
            console.log('Nessun voto disponibile.');
        }
        },
        // Metodo per arrotondare un numero alla metà più vicina
        roundToHalf(num) {
        return Math.round(num * 2) / 2;
        },
        resetForm() {
        this.contactForm = {
            firstname: '',
            lastname: '',
            email: '',
            message: '',
            user_id: this.singleMusician.id
        };
        },
        toggleReviewForm() {
            this.showReviewForm = !this.showReviewForm;
        },
        closeReviewForm() {
            this.showReviewForm = false;
        },
        setVote(star) {
            this.selectedStars = star; // Imposta la stella attiva al passaggio del mouse
        },
        setActiveStar(star) {
            if (!this.voteForm.vote) {
                this.activeStar = star; // Imposta la stella attiva solo se il voto non è stato già assegnato
            }
        },
        clearMessages() {
            this.errorMessage = '';
            this.successMessage = '';
        },
        openOffcanvas() {
        // Open the offcanvas
            this.showOffcanvas = true;
        },
        closeOffcanvas() {
        // Close the offcanvas
            this.showOffcanvas = false;
        }
    },
    created() {
        this.fetchUserData();
    },
    mounted() {
    this.showContactForm = false; // Nascondi il form dei contatti all'inizio
    window.scrollTo(0, 0); // Scorri verso l'alto quando il componente viene montato

    // Mostra il form dei contatti dopo un breve ritardo
    setTimeout(() => {
        this.showContactForm = true;
    }, 500); // Tempo in millisecondi prima di mostrare il form dei contatti
}
};
</script>


<template>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.1/font/bootstrap-icons.css" integrity="sha384-4LISF5TTJX/fLmGSxO53rV4miRxdg84mZsxmO8Rx5jGtp/LbrixFETvWa5a6sESd" crossorigin="anonymous">
    <!-- Sezione con tutte le informazioni del musicista -->
    <section>
        <!-- Immagine e informazioni sul musicista -->
        <div v-if="singleMusician && singleMusician.user_details" class="container-fluid user-img d-flex justify-content-center flex-column mb-5 bounce-in"
            :style="{ 'background-image': 'url(http://127.0.0.1:8000/storage/' + singleMusician.user_details.picture + ')' }"
        >
           <div class="container">
                <!-- Nome e città -->
                <h1 class="text-white pb-3 bounce-in-x display-1 fw-bold">{{ singleMusician.name }}</h1>
                <h2 class="text-white bounce-in-x display-2 fw-bold d-none d-md-block">{{ singleMusician.city }}</h2>
           </div>
        </div>
        <!-- Altre informazioni sul musicista -->
        <div class="container mb-5">
            <div class="row justify-content-between">
            <!-- Bio -->
            <div class="col-6 biography-section">
                <h2 class="fw-bold pt-3 mb-4">Bio</h2>
                <p v-if="singleMusician && singleMusician.user_details.bio">{{ singleMusician.user_details.bio }}</p>
                <p v-else>Ancora nessun dato inserito dall'utente.</p>
            </div>
            <!-- Competenze -->
            <div class="col-4 text-end skills-container d-none d-md-block">
                <h2 class="fw-bold pt-3 mb-4">Competenze</h2>
                <div v-if="singleMusician && singleMusician.roles.length > 0">
                    <span v-for="(role, i) in singleMusician.roles" :key="i" class="bounce-in">
                        <img :src="'http://127.0.0.1:8000/storage/' + role.icon" alt="roleInstrument" class="instrument-pic p-lg-1 p-sm-3">
                    </span>
                </div>
                <p v-else>Ancora nessun dato inserito dall'utente.</p>
            </div>
            <!-- Demo -->
            <div class="col-12 mb-5">
                <h2 class="fw-bold pt-3 mb-4">Demo</h2>
                <audio v-if="singleMusician && singleMusician.user_details.demo" controls class="w-100">
                    <source :src="demoPath" type="audio/mpeg">
                </audio>
                <p v-else>Ancora nessun dato inserito dall'utente.</p>
            </div>
            <!-- Info di contatto -->
            <div class="col-12 mb-5">
                <h2 class="fw-bold pt-3 mb-4">Info di contatto</h2>
                <div v-if="singleMusician && singleMusician" class="">
                    <i class="fa-solid fa-envelope me-3"></i>Mail:<span class="ps-3">{{ singleMusician.email }}</span>
                </div>
                <p v-else>Ancora nessun dato inserito dall'utente.</p>

                <div v-if="singleMusician && singleMusician.user_details">
                    <i class="fa-solid fa-phone me-3"></i>Cell: 
                    <span  v-if="singleMusician && singleMusician.user_details.cellphone" class="ps-3">
                          {{ singleMusician.user_details.cellphone }}
                    </span>
                    <span class="ps-3" v-else><small>Numero di telefono non inserito</small></span>
                </div>
                
            </div>
        </div>
        <!-- ------------------------------------------------------------------------------------------------------------------------------------ -->
       <section>
            <div v-if="singleMusician && singleMusician.reviews" class="d-flex justify-content-between mb-5 padding">
                <h2 class="mb-5 fw-bold">Recensioni:</h2>
                <div class="hover-container">
                    <div class="d-flex align-items-center">
                        <!-- Numero totale delle recensioni -->
                        <small class="me-2 number-reviews">{{ singleMusician.votes.length }} voti</small>
                        <!-- Visualizza le palline riempite in base alla media dei voti -->
                        <span
                            v-for="index in 5"
                            :key="index"
                            :class="{
                                'filled': index <= averageVote,
                                'half-filled': index === Math.ceil(averageVote) && averageVote % 1 !== 0
                            }"
                            class="vote-star"
                        >
                            <!-- Utilizzo delle icone di Font Awesome -->
                            <i v-if="index <= averageVote" class="fa-solid fa-circle"></i>
                            <i v-else class="fa-regular fa-circle"></i>
                        </span>
                    </div>
                </div>
            </div>
            <div v-if="singleMusician && singleMusician.reviews && singleMusician.reviews.length > 0" class="mb-5">
                <swiper
                    :effect="'coverflow'"
                    :grabCursor="true"
                    :centeredSlides="true"
                    :slidesPerView="'auto'"
                    :coverflowEffect="{
                    rotate: 50,
                    stretch: 0,
                    depth: 100,
                    modifier: 1,
                    slideShadows: false,
                    }"
                    :pagination="false"
                    :modules="modules"
                    :initialSlide="initialSlideIndex" 
                    class="mySwiper"
                >
                    <swiper-slide  v-for="(review, index) in singleMusician.reviews" :key="index">
                        <div class="card mb-3">
                            <div class="card-body px-4 pt-4">
                                <h5 class="card-title fs-3 mb-4">{{ review.firstname + ' ' + review.lastname }}</h5>
                                <p class="card-text">{{ review.description }}</p>
                            </div>
                        </div>
                    </swiper-slide>
                </swiper>
            </div>
            <div v-else class="mb-5">
                Non ci sono recensioni
            </div>
            <div class="d-flex justify-content-center margin">
                <button @click="toggleReviewForm" v-if="!showReviewForm" class="btn-write-review">Scrivi una recensione</button>
            </div>
       </section>
        


        <!-- ------------------------------------------------------------------------------------------------------------------------------------ -->

            
        </div>
    </section>


    <!-- Sezione per scrivere una recensione -->
    
    <section v-if="showReviewForm" id="sectionReviews" class="contact-section" style="background-color: white;">
        <div class="container">
            <h2 class="text-black text-center py-5">Lascia qui la tua opinione!</h2>
            <transition name="slide-down">
                <form @submit.prevent="submitReview" class="contact-form">
                    <!-- Input per nome -->
                    <div class="mb-3">
                        <label for="review-firstname" class="form-label text-black">Nome</label>
                        <input type="text" v-model="reviewForm.firstname" class="form-control" :class="{ 'is-invalid': showReviewFormError && !reviewForm.firstname && reviewForm.description && !voteForm.vote }" id="review-firstname" placeholder="Inserisci il tuo nome..." maxlength="74">
                        <div class="invalid-feedback">Il nome è richiesto.</div>
                    </div>
                    <!-- Input per cognome -->
                    <div class="mb-3">
                        <label for="review-lastname" class="form-label text-black">Cognome</label>
                        <input type="text" v-model="reviewForm.lastname" class="form-control" :class="{ 'is-invalid': showReviewFormError && !reviewForm.lastname && reviewForm.description && !voteForm.vote }" id="review-lastname" placeholder="Inserisci il tuo cognome..." maxlength="74">
                        <div class="invalid-feedback">Il cognome è richiesto.</div>
                    </div>
                    <!-- Input per recensione -->
                    <div class="mb-3">
                        <label for="description" class="form-label text-black">Recensione</label>
                        <textarea v-model="reviewForm.description" class="form-control" id="description" rows="5" maxlength="2048" placeholder="Inserisci la tua recensione..."></textarea>
                    </div>
                    <!-- Input per voto -->
                    <div class="mb-3">
                        <label class="form-label text-black">Voto:</label>
                        <div class="rating-input">
                            <i v-for="n in 5" :key="n" class="fa-regular fa-circle fa-xl me-2" :class="{ 'fas': n <= voteForm.vote }" @click="voteForm.vote = n"></i>
                        </div>
                    </div>
                    <div class="text-center py-5 d-flex align-item-center justify-content-center">
                        <button type="submit" class="btn my-button btn-lg me-5 text-white px-4" style="background-color: #21252b;">Invia</button>
                        <button type="button" @click="closeReviewForm" class="my-button btn-lg me-5 text-white px-4 fw-bold" style="background-color: #21252b;">Chiudi</button>
                    </div>
                </form>
            </transition>
        </div>
    </section>

    <!-- Sezione Contatti -->
    <section class="contact-section">
            <div class="container">
                <h2 class="text-white text-center py-5">Contatta l'utente!</h2>
                <form @submit.prevent="sendMessage" novalidate>
                    <!-- Input per nome -->
                    <div class="mb-3">
                        <label for="firstname" class="form-label text-white">Nome*</label>
                        <input type="text" v-model="contactForm.firstname" class="form-control" :class="{ 'is-invalid': showContactFormError && !contactForm.firstname }" id="firstname" placeholder="Inserisci il tuo nome..." required maxlength="74" aria-describedby="firstname-error">
                        <div id="firstname-error" class="invalid-feedback">Il nome è richiesto.</div>
                    </div>
                    <!-- Input per cognome -->
                    <div class="mb-3">
                        <label for="lastname" class="form-label text-white">Cognome*</label>
                        <input type="text" v-model="contactForm.lastname" class="form-control" :class="{ 'is-invalid': showContactFormError && !contactForm.lastname }" id="lastname" placeholder="Inserisci il tuo cognome..." required maxlength="74" aria-describedby="lastname-error">
                        <div id="lastname-error" class="invalid-feedback">Il cognome è richiesto.</div>
                    </div>
                    <!-- Input per email -->
                    <div class="mb-3">
                        <label for="email" class="form-label text-white">Email*</label>
                        <input type="email" v-model="contactForm.email" class="form-control" :class="{ 'is-invalid': showContactFormError && !contactForm.email }" id="email" placeholder="Inserisci la tua mail..." required maxlength="255" aria-describedby="email-error">
                        <div id="email-error" class="invalid-feedback">L'email è richiesta.</div>
                    </div>
                    <!-- Input per messaggio -->
                    <div class="mb-3">
                        <label for="message" class="form-label text-white">Messaggio*</label>
                        <textarea v-model="contactForm.message" class="form-control" :class="{ 'is-invalid': showContactFormError && !contactForm.message }" id="message" rows="5" required maxlength="2048" placeholder="Lascia un messaggio che verrà visualizzato dall'artista..." aria-describedby="message-error"></textarea>
                        <div id="message-error" class="invalid-feedback">Il messaggio è richiesto.</div>
                    </div>
                    <div class="text-center py-5">
                        <button type="submit" class="btn my-button btn-lg px-5">Invia</button>
                    </div>
                </form>
            </div>
        </section>


        <section class="message-section">
            <div v-if="reviewErrorMessage || successMessage" class="message-container d-sm-flex" :class="{ active: reviewErrorMessage || successMessage }">
                <div v-if="reviewErrorMessage" class="alert alert-danger" role="alert">
                    <span class="icon"><i class="bi bi-exclamation-triangle-fill"></i></span>
                    <span>{{ reviewErrorMessage }}</span>
                    <button @click="clearMessages" class="btn-close">&times;</button>
                </div>
                <div v-if="successMessage" class="alert alert-success" role="alert">
                    <span class="icon"><i class="bi bi-check-circle-fill"></i></span>
                    <span>{{ successMessage }}</span>
                    <button @click="clearMessages" class="btn-close">&times;</button>
                </div>
            </div>

            <div v-if="contactErrorMessage || successMessage" class="message-container d-sm-flex" :class="{ active: contactErrorMessage || successMessage }">
                <div v-if="contactErrorMessage" class="alert alert-danger" role="alert">
                    <span class="icon"><i class="bi bi-exclamation-triangle-fill"></i></span>
                    <span>{{ contactErrorMessage }}</span>
                    <button @click="clearMessages" class="btn-close">&times;</button>
                </div>
                <div v-if="successMessage" class="alert alert-success" role="alert">
                    <span class="icon"><i class="bi bi-check-circle-fill"></i></span>
                    <span>{{ successMessage }}</span>
                    <button @click="clearMessages" class="btn-close">&times;</button>
                </div>
            </div>
        </section>
</template>

<style lang="scss" scoped>
@import '../assets/scss/partials/ProfilePage.scss';


.hover-container .number-reviews {
    opacity: 0;
    transition: opacity 0.5s ease;
}

.hover-container:hover .number-reviews {
    opacity: 1;
}

.instrument-pic {
    max-width: 75px;
    max-height: 75px;
    margin: 0 30px;
}

.slide-down-enter-active {
    transition: all 0.5s ease;
}
.slide-down-enter, .slide-down-leave-to {
    transform: translateY(-100%);
    opacity: 0;
}

.message-section {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 1000;
}

.message-container {
    width: 75%;
    margin: 0 auto;
    max-width: 500px;
    margin-bottom: 10px;
    opacity: 0;
    transition: opacity 0.5s ease;
}

.message-container.active {
    opacity: 1;
}

.alert {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 15px;
    border-radius: 10px;
    margin-bottom: 10px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    background: linear-gradient(to right, #ff416c, #ff4b2b);
    color: #ffffff;
}

.alert span {
    flex-grow: 1;
    margin-right: 10px;
}

.alert-danger {
    background: linear-gradient(to right, #ed213a, #93291e);
}

.alert-success {
    background: linear-gradient(to right, #11998e, #38ef7d);
}

.icon {
    margin-right: 10px;
    font-size: 24px;
}

.btn-close {
    cursor: pointer;
    background: none;
    border: none;
    font-size: 18px;
    line-height: 1;
    color: #ffffff;
    transition: color 0.3s ease;
}

.btn-close:hover {
    color: #cccccc;
}

.vote-container {
    position: relative; /* Assicura che gli elementi posizionati all'interno rispettino questo contenitore */
    overflow: hidden; /* Nasconde gli elementi che escono dai confini del contenitore */
}

.filled {
    color: #21252b; /* Colore per le palline piene */
    margin-right: 3px;
}

.fa-circle:before {
    margin-right: 3px;
}

.half-filled {
    position: relative; /* Posizione relativa per le palline parzialmente piene */
}

.half-filled::before {
    content: '\f111'; /* Unicode per l'icona della pallina piena di Font Awesome */
    overflow: hidden;
    margin-right: 5px;
    z-index: 20;
    font-family: 'Font Awesome 5 Free'; /* Font family per Font Awesome */
    position: absolute; /* Posizione assoluta rispetto al contenitore */
    color: transparent; /* Colore trasparente per l'icona della pallina piena */
    width: 50%; /* Larghezza del contenitore per l'icona */
    white-space: nowrap; /* Evita il wrap del testo */
}

.half-filled::after {
    content: "";
    position: absolute;
    overflow: hidden;
    z-index: 20;
    top: .3px;
    left: -1px;
    width: 50%;
    height: 25px;
    background-color: #21252b;
    display: inline-block;
    border-radius: 30px 0 0 30px;
    margin-top: 5px;
}
.card{
    border: none;
}
.card-body{
    background-color: #21252b;
    color: white;
    border-radius: 30px;
    border: 0 solid white;
    min-height: 350px
}

.swiper {
  width: 100%;
  padding-top: 50px;
  padding-bottom: 50px;
}

.swiper-slide {
  background-position: center;
  background-size: cover;
  width: 300px;
  height: 300px;
}

.swiper-slide img {
  display: block;
  width: 100%;
}

@media (max-width: 425px) {
    .user-img {
        min-height: 300px;
    }
    .skills-container {
        display: none;
    }
    .biography-section {
        width: 100%;
    }
    .mb-5 {
        margin: 0 !important;
    }
    .padding {
        padding-top: 20px;
        padding-bottom: 20px; 
    }
    .margin {
        margin-bottom: 15px;
    }
    .card {
        min-height: 200px;
    }
}

@media (max-width: 767px) {
    
    .biography-section {
        width: 100%;
    }
    .skills-container {
        display: none;
    }
}
</style>