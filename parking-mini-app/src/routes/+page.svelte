<script>
    import { onMount } from "svelte";
    import { fade, fly } from "svelte/transition";

    let authCode = $state("");
    let token = $state("");
    let parkingSpot = $state("");
    let parkingHours = $state(1);
    let totalCost = $derived(parkingHours * 1000);

    // Steps: 'login' -> 'scan' -> 'details' -> 'paid'
    let currentStep = $state("login");

    function authenticate() {
        // @ts-ignore
        if (typeof my !== "undefined") {
            // @ts-ignore
            my.getAuthCode({
                scopes: ["auth_base", "USER_ID"],
                success: (/** @type {{ authCode: string; }} */ res) => {
                    authCode = res.authCode;

                    fetch("https://its.mouamle.space/api/auth-with-superQi", {
                        method: "POST",
                        headers: {
                            "Content-Type": "application/json",
                        },
                        body: JSON.stringify({
                            token: authCode,
                        }),
                    })
                        .then((res) => res.json())
                        .then((data) => {
                            token = data.token;
                            // @ts-ignore
                            my.alert({
                                content: "Login successful",
                                buttonText: "Continue",
                            });
                            currentStep = "scan";
                        })
                        .catch((err) => {
                            let errorDetails = "";
                            if (err && typeof err === "object") {
                                errorDetails = JSON.stringify(err, null, 2);
                            } else {
                                errorDetails = String(err);
                            }
                            // @ts-ignore
                            my.alert({
                                content: "Login Error: " + errorDetails,
                            });
                        });
                },
                fail: (/** @type {{ authErrorScopes: any; }} */ res) => {
                    console.log(res.authErrorScopes);
                },
            });
        } else {
            // Fallback for browser testing
            // Using a slight delay to feel more natural
            setTimeout(() => {
                currentStep = "scan";
                token = "mock-token";
            }, 800);
        }
    }

    function scan() {
        // @ts-ignore
        if (typeof my !== "undefined") {
            // @ts-ignore
            my.scan({
                type: "qr",
                success: (/** @type {{ code: any; }} */ res) => {
                    parkingSpot = res.code;
                    currentStep = "details";
                    // @ts-ignore
                    my.alert({ title: "Found Spot: " + res.code });
                },
            });
        } else {
            // Fallback for browser testing
            setTimeout(() => {
                let mockCode = prompt(
                    "Simulate QR Scan (Enter Spot ID):",
                    "ZONE-A-01",
                );
                if (mockCode) {
                    parkingSpot = mockCode;
                    currentStep = "details";
                }
            }, 500);
        }
    }

    function pay() {
        // @ts-ignore
        if (typeof my !== "undefined") {
            fetch("https://its.mouamle.space/api/payment", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    Authorization: token,
                },
            })
                .then((res) => res.json())
                .then((data) => {
                    // @ts-ignore
                    my.tradePay({
                        paymentUrl: data.url,
                        success: (/** @type {any} */ res) => {
                            currentStep = "paid";
                            // @ts-ignore
                            my.alert({
                                content: "Payment successful",
                            });
                        },
                    });
                })
                .catch((err) => {
                    // @ts-ignore
                    my.alert({
                        content: "Payment failed. Please try again.",
                    });
                });
        } else {
            // Fallback for browser testing
            setTimeout(() => {
                currentStep = "paid";
            }, 1500);
        }
    }

    function reset() {
        currentStep = "login";
        parkingSpot = "";
        parkingHours = 1;
        authCode = "";
        token = "";
    }
</script>

<div class="min-h-screen bg-slate-50 font-sans text-slate-800 flex flex-col">
    <!-- Header -->
    <header
        class="bg-white/80 backdrop-blur-md border-b border-slate-200 sticky top-0 z-20 px-6 py-4 flex items-center justify-center shadow-sm"
    >
        <h1
            class="text-lg font-bold bg-gradient-to-r from-blue-600 to-indigo-600 bg-clip-text text-transparent"
        >
            SmartParking
        </h1>
    </header>

    <!-- Main Content -->
    <main
        class="flex-1 flex flex-col items-center justify-center p-6 w-full max-w-md mx-auto"
    >
        <!-- Step 1: Login -->
        {#if currentStep === "login"}
            <div
                in:fly={{ y: 20, duration: 400 }}
                class="flex flex-col items-center gap-6 w-full text-center"
            >
                <div
                    class="w-20 h-20 bg-blue-100 rounded-full flex items-center justify-center text-blue-600 mb-2"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-10 w-10"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M11 16l-4-4m0 0l4-4m-4 4h14m-5 4v1a3 3 0 01-3 3H6a3 3 0 01-3-3V7a3 3 0 013-3h7a3 3 0 013 3v1"
                        />
                    </svg>
                </div>
                <div class="space-y-2">
                    <h2 class="text-2xl font-bold text-slate-900">
                        Welcome Back
                    </h2>
                    <p class="text-slate-500 text-sm leading-relaxed">
                        Login with your SuperQi account to easy park and pay.
                    </p>
                </div>

                <button
                    class="w-full bg-blue-600 text-white font-semibold py-4 rounded-2xl shadow-lg shadow-blue-600/20 active:scale-[0.98] transition-all hover:bg-blue-700 mt-4 cursor-pointer"
                    onclick={authenticate}
                >
                    Login with SuperQi
                </button>
            </div>
        {/if}

        <!-- Step 2: Scan -->
        {#if currentStep === "scan"}
            <div
                in:fly={{ y: 20, duration: 400 }}
                class="flex flex-col items-center gap-6 w-full text-center"
            >
                <div
                    class="w-20 h-20 bg-indigo-100 rounded-full flex items-center justify-center text-indigo-600 mb-2 animate-pulse"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-10 w-10"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h.01M12 12h4.01M16 20h4M4 12h4m12 0h.01M5 8h2a1 1 0 001-1V5a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1zm12 0h2a1 1 0 001-1V5a1 1 0 00-1-1h-2a1 1 0 00-1 1v2a1 1 0 001 1zM5 20h2a1 1 0 001-1v-2a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1z"
                        />
                    </svg>
                </div>
                <div class="space-y-2">
                    <h2 class="text-2xl font-bold text-slate-900">
                        Find Your Spot
                    </h2>
                    <p class="text-slate-500 text-sm leading-relaxed">
                        Locate the QR code on your parking pillar and scan it.
                    </p>
                </div>

                <button
                    class="w-full bg-indigo-600 text-white font-semibold py-4 rounded-2xl shadow-lg shadow-indigo-600/20 active:scale-[0.98] transition-all hover:bg-indigo-700 mt-4 cursor-pointer"
                    onclick={scan}
                >
                    Scan QR Code
                </button>
            </div>
        {/if}

        <!-- Step 3: Details & Pay -->
        {#if currentStep === "details"}
            <div
                in:fly={{ y: 20, duration: 400 }}
                class="flex flex-col gap-6 w-full"
            >
                <div class="text-center space-y-1">
                    <h2 class="text-2xl font-bold text-slate-900">
                        Confirm Parking
                    </h2>
                    <p class="text-slate-500 text-sm">
                        Review details before payment
                    </p>
                </div>

                <div
                    class="bg-white rounded-3xl p-6 shadow-sm border border-slate-100 flex flex-col gap-4"
                >
                    <div
                        class="flex items-center justify-between p-3 bg-slate-50 rounded-xl"
                    >
                        <span class="text-slate-500 text-sm font-medium"
                            >Spot ID</span
                        >
                        <span class="text-slate-900 font-bold tracking-wide"
                            >{parkingSpot}</span
                        >
                    </div>

                    <div class="flex flex-col gap-2">
                        <label
                            class="text-slate-500 text-sm font-medium ml-1"
                            for="hours">Duration</label
                        >
                        <div class="relative">
                            <input
                                id="hours"
                                type="number"
                                min="1"
                                max="24"
                                bind:value={parkingHours}
                                class="w-full p-4 bg-slate-50 border-none rounded-2xl text-lg font-bold text-center focus:ring-2 focus:ring-blue-500 transition-all outline-none"
                            />
                            <span
                                class="absolute right-4 top-1/2 -translate-y-1/2 text-slate-400 font-medium pointer-events-none"
                                >Hours</span
                            >
                        </div>
                    </div>

                    <div
                        class="py-4 border-t border-dashed border-slate-200 mt-2 space-y-3"
                    >
                        <div class="flex justify-between items-center text-sm">
                            <span class="text-slate-500">Rate</span>
                            <span class="font-medium text-slate-700"
                                >1,000 IQD / hr</span
                            >
                        </div>
                        <div class="flex justify-between items-center">
                            <span class="text-slate-900 font-bold text-lg"
                                >Total Cost</span
                            >
                            <span class="text-2xl font-black text-blue-600"
                                >{totalCost.toLocaleString()}
                                <span class="text-sm font-bold text-blue-400"
                                    >IQD</span
                                ></span
                            >
                        </div>
                    </div>
                </div>

                <button
                    class="w-full bg-green-600 text-white font-semibold py-4 rounded-2xl shadow-lg shadow-green-600/20 active:scale-[0.98] transition-all hover:bg-green-700 cursor-pointer"
                    onclick={pay}
                >
                    Pay & Park
                </button>
            </div>
        {/if}

        <!-- Step 4: Success -->
        {#if currentStep === "paid"}
            <div
                in:scale={{ duration: 400, start: 0.9 }}
                class="flex flex-col items-center gap-6 w-full text-center max-w-sm"
            >
                <div
                    class="w-24 h-24 bg-green-100 rounded-full flex items-center justify-center text-green-500 mb-2 shadow-sm"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-12 w-12"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="3"
                            d="M5 13l4 4L19 7"
                        />
                    </svg>
                </div>
                <div class="space-y-3">
                    <h2 class="text-3xl font-bold text-slate-900">Success!</h2>
                    <p class="text-slate-500 text-sm leading-relaxed">
                        Your parking has been confirmed.
                    </p>
                </div>

                <div
                    class="bg-slate-50 rounded-2xl p-6 w-full mt-2 border border-slate-100"
                >
                    <div class="flex justify-between mb-2">
                        <span class="text-slate-400 text-sm">Amount Paid</span>
                        <span class="font-bold text-slate-700"
                            >{totalCost.toLocaleString()} IQD</span
                        >
                    </div>
                    <div class="flex justify-between">
                        <span class="text-slate-400 text-sm">Duration</span>
                        <span class="font-bold text-slate-700"
                            >{parkingHours} Hours</span
                        >
                    </div>
                </div>

                <button
                    class="w-full bg-slate-900 text-white font-medium py-3 rounded-xl shadow-lg shadow-slate-900/10 active:scale-[0.98] transition-all hover:bg-slate-800 mt-6 cursor-pointer"
                    onclick={reset}
                >
                    Back to Home
                </button>
            </div>
        {/if}
    </main>
</div>

<style>
    /* Custom utility overrides if needed, though Tailwind covers most */
    input[type="number"]::-webkit-inner-spin-button,
    input[type="number"]::-webkit-outer-spin-button {
        -webkit-appearance: none;
        margin: 0;
    }
</style>
