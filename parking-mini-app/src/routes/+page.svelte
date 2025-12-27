<script>
    import { onMount } from "svelte";

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
                                content: "Error: " + errorDetails,
                            });
                        });
                },
                fail: (/** @type {{ authErrorScopes: any; }} */ res) => {
                    console.log(res.authErrorScopes);
                },
            });
        } else {
            // Fallback for browser testing without Hylid
            alert("Hylid environment not found. Mocking login success.");
            currentStep = "scan";
            token = "mock-token";
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
                    // @ts-ignore
                    my.alert({ title: "Scanned: " + res.code });
                    currentStep = "details";
                },
            });
        } else {
            // Fallback for browser testing
            let mockCode = prompt(
                "Enter mock QR code (e.g., SPOT-123):",
                "SPOT-123",
            );
            if (mockCode) {
                parkingSpot = mockCode;
                currentStep = "details";
            }
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
                            // @ts-ignore
                            my.alert({
                                content: "Payment successful",
                            });
                            currentStep = "paid";
                        },
                    });
                })
                .catch((err) => {
                    // @ts-ignore
                    my.alert({
                        content: "Payment failed",
                    });
                });
        } else {
            // Fallback for browser testing
            alert(`Simulating Payment of ${totalCost} IQD`);
            currentStep = "paid";
        }
    }

    function reset() {
        currentStep = "login";
        parkingSpot = "";
        parkingHours = 1;
        authCode = "";
        token = "";
    }

    function copyAuthCode() {
        if (authCode) {
            navigator.clipboard.writeText(authCode);
            alert("Auth code copied!");
        } else {
            alert("No auth code available yet.");
        }
    }
</script>

<div
    class="h-16 w-full bg-stone-700 flex items-center justify-start px-4 fixed top-0 left-0 z-10"
>
    <h1 class="text-2xl font-bold text-white">Parking MiniApp</h1>
</div>

<div
    class="w-full h-screen flex flex-col items-center gap-6 justify-start p-4 mt-16"
>
    <!-- Step 1: Login -->
    {#if currentStep === "login"}
        <div class="flex flex-col items-center gap-4 w-full max-w-sm">
            <h2 class="text-xl font-semibold">Welcome</h2>
            <p class="text-gray-600 text-center">
                Please log in to start parking.
            </p>
            <button
                class="bg-blue-500 text-white px-6 py-3 rounded-xl w-full font-medium shadow-lg active:scale-95 transition-all cursor-pointer"
                onclick={authenticate}
            >
                Login with SuperQi
            </button>
        </div>
    {/if}

    <!-- Step 2: Scan -->
    {#if currentStep === "scan"}
        <div class="flex flex-col items-center gap-4 w-full max-w-sm">
            <h2 class="text-xl font-semibold">Scan Parking Spot</h2>
            <p class="text-gray-600 text-center">
                Scan the QR code at your parking spot.
            </p>
            <button
                class="bg-blue-500 text-white px-6 py-3 rounded-xl w-full font-medium shadow-lg active:scale-95 transition-all cursor-pointer"
                onclick={scan}
            >
                Scan QR Code
            </button>
            {#if authCode}
                <button
                    class="text-sm text-gray-400 underline"
                    onclick={copyAuthCode}>Copy Debug Auth Code</button
                >
            {/if}
        </div>
    {/if}

    <!-- Step 3: Details & Pay -->
    {#if currentStep === "details"}
        <div class="flex flex-col items-center gap-4 w-full max-w-sm">
            <h2 class="text-xl font-semibold">Parking Details</h2>

            <div class="bg-gray-100 p-4 rounded-lg w-full">
                <p class="text-sm text-gray-500">Spot ID</p>
                <p class="text-lg font-bold">{parkingSpot}</p>
            </div>

            <div class="w-full">
                <label class="block text-sm text-gray-500 mb-1" for="hours"
                    >Duration (Hours)</label
                >
                <input
                    id="hours"
                    type="number"
                    min="1"
                    bind:value={parkingHours}
                    class="w-full p-3 border border-gray-300 rounded-lg text-lg text-center"
                />
            </div>

            <div class="flex justify-between w-full items-center p-2">
                <span class="text-gray-600">Rate</span>
                <span class="font-medium">1,000 IQD / hr</span>
            </div>

            <div class="w-full border-t border-gray-200 my-2"></div>

            <div class="flex justify-between w-full items-center">
                <span class="text-xl font-bold">Total</span>
                <span class="text-xl font-bold text-blue-600"
                    >{totalCost.toLocaleString()} IQD</span
                >
            </div>

            <button
                class="bg-green-500 text-white px-6 py-3 rounded-xl w-full font-medium shadow-lg active:scale-95 transition-all mt-4 cursor-pointer"
                onclick={pay}
            >
                Pay Now
            </button>
        </div>
    {/if}

    <!-- Step 4: Success -->
    {#if currentStep === "paid"}
        <div
            class="flex flex-col items-center gap-4 w-full max-w-sm text-center"
        >
            <div
                class="w-16 h-16 bg-green-100 text-green-500 rounded-full flex items-center justify-center text-3xl mb-2"
            >
                ✓
            </div>
            <h2 class="text-2xl font-bold text-green-600">
                Payment Successful!
            </h2>
            <p class="text-gray-600">
                You have paid <b>{totalCost.toLocaleString()} IQD</b> for
                <b>{parkingHours} hours</b>.
            </p>

            <button
                class="bg-gray-800 text-white px-6 py-3 rounded-xl w-full font-medium shadow-lg active:scale-95 transition-all mt-8 cursor-pointer"
                onclick={reset}
            >
                Done
            </button>
        </div>
    {/if}
</div>
