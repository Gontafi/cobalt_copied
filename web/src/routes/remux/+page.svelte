<script lang="ts">
    import mime from "mime";
    import LibAVWrapper from "$lib/libav";
    import { beforeNavigate, goto } from "$app/navigation";

    import { t } from "$lib/i18n/translations";
    import { createDialog } from "$lib/state/dialogs";
    import { downloadFile } from "$lib/download";

    import Skeleton from "$components/misc/Skeleton.svelte";
    import DropReceiver from "$components/misc/DropReceiver.svelte";
    import FileReceiver from "$components/misc/FileReceiver.svelte";
    import BulletExplain from "$components/misc/BulletExplain.svelte";

    import IconRepeat from "@tabler/icons-svelte/IconRepeat.svelte";
    import IconDevices from "@tabler/icons-svelte/IconDevices.svelte";
    import IconInfoCircle from "@tabler/icons-svelte/IconInfoCircle.svelte";

    let draggedOver = false;
    let file: File | undefined;

    let totalDuration: number | undefined;
    let processedDuration: number | undefined;

    let speed: number | undefined;
    let progress: string | undefined;

    let wentAway = false;

    $: {
        if (totalDuration && processedDuration) {
            const percentage = Math.max(
                0,
                Math.min(100, (processedDuration / totalDuration) * 100)
            ).toFixed(2);
            progress = percentage;
        } else if (processing) {
            progress = undefined;
            speed = undefined;
        } else {
            progress = undefined;
            speed = undefined;
        }
    }

    let processing = false;

    const ff = new LibAVWrapper((progress) => {
        if (progress.out_time_sec) {
            processedDuration = progress.out_time_sec;
        }

        if (progress.speed) {
            speed = progress.speed;
        }
    });

    ff.init();

    const render = async () => {
        if (!file || processing) return;
        await ff.init();

        let dialogOpened;
        try {
            progress = undefined;
            speed = undefined;
            processing = true;

            const file_info = await ff.probe(file).catch((e) => {
                if (e?.message?.toLowerCase().includes("out of memory")) {
                    console.error("uh oh! out of memory");
                    console.error(e);

                    dialogOpened = true;
                    return createDialog({
                        id: "remux-error",
                        type: "small",
                        meowbalt: "error",
                        bodyText: $t("error.remux.out_of_resources"),
                        buttons: [
                            {
                                text: $t("button.gotit"),
                                main: true,
                                action: () => {},
                            },
                        ],
                    });
                }
            });

            if (!file_info?.format) {
                if (!dialogOpened)
                    return createDialog({
                        id: "remux-error",
                        type: "small",
                        meowbalt: "error",
                        bodyText: $t("error.remux.corrupted"),
                        buttons: [
                            {
                                text: $t("button.gotit"),
                                main: true,
                                action: () => {},
                            },
                        ],
                    });
                return;
            }

            totalDuration = Number(file_info.format.duration);

            if (file instanceof File && !file.type) {
                file = new File([file], file.name, {
                    type: mime.getType(file.name) ?? undefined,
                });
            }

            const render = await ff
                .render({
                    blob: file,
                    args: ["-c", "copy", "-map", "0"],
                })
                .catch((e) => {
                    console.error("uh-oh! render error");
                    console.error(e);
                    return createDialog({
                        id: "remux-error",
                        type: "small",
                        meowbalt: "error",
                        bodyText: $t("error.remux.out_of_resources"),
                        buttons: [
                            {
                                text: $t("button.gotit"),
                                main: true,
                                action: () => {},
                            },
                        ],
                    });
                });

            if (!render) {
                return console.log("not a valid file");
            }

            const filenameParts = file.name.split(".");
            const filenameExt = filenameParts.pop();

            const filename = `${filenameParts.join(".")} (remux).${filenameExt}`;

            return await downloadFile({
                file: new File([render], filename, {
                    type: render.type,
                }),
            });
        } finally {
            processing = false;
            file = undefined;
            progress = undefined;
            speed = undefined;
        }
    };

    beforeNavigate((event) => {
        if (processing && !wentAway) {
            event.cancel();
            const path = event.to?.route?.id;

            if (path) {
                return createDialog({
                    id: "remux-ongoing",
                    type: "small",
                    icon: "warn-red",
                    title: $t("dialog.processing.title.ongoing"),
                    bodyText: $t("dialog.processing.ongoing"),
                    buttons: [
                        {
                            text: $t("button.no"),
                            main: false,
                            action: () => {},
                        },
                        {
                            text: $t("button.yes"),
                            main: true,
                            color: "red",
                            action: async () => {
                                await ff.terminate();
                                wentAway = true;
                                goto(path);
                            },
                        },
                    ],
                });
            }
        }
    });

    $: if (file) {
        render();
    }
</script>

<svelte:head>
    <title>{$t("tabs.remux")} ~ {$t("general.cobalt")}</title>
    <meta
        property="og:title"
        content="{$t('tabs.remux')} ~ {$t('general.cobalt')}"
    />
</svelte:head>

<DropReceiver
    bind:file
    bind:draggedOver
    id="remux-container"
    classes={processing ? "processing" : ""}
>
    <div
        id="remux-open"
        class:processing
        tabindex="-1"
        data-first-focus
        data-focus-ring-hidden
    >
        <div id="remux-receiver">
            <FileReceiver
                bind:draggedOver
                bind:file
                acceptTypes={["video/*", "audio/*"]}
                acceptExtensions={[
                    "mp4",
                    "webm",
                    "mp3",
                    "ogg",
                    "opus",
                    "wav",
                    "m4a",
                ]}
            />
        </div>

        <div id="remux-bullets">
            <BulletExplain
                title={$t("remux.bullet.purpose.title")}
                description={$t("remux.bullet.purpose.description")}
                icon={IconRepeat}
            />

            <BulletExplain
                title={$t("remux.bullet.explainer.title")}
                description={$t("remux.bullet.explainer.description")}
                icon={IconInfoCircle}
            />

            <BulletExplain
                title={$t("remux.bullet.privacy.title")}
                description={$t("remux.bullet.privacy.description")}
                icon={IconDevices}
            />
        </div>
    </div>

    <div id="remux-processing" class:processing aria-hidden={!processing}>
        <div id="processing-status">
            {#if processing}
                {#if progress && speed}
                    <div class="progress-bar">
                        <Skeleton
                            width="{progress}%"
                            height="20px"
                            class="elevated"
                        />
                    </div>
                    <div class="progress-text">
                        processing ({progress}%, {speed}x)...
                    </div>
                {:else}
                    processing...
                {/if}
            {:else}
                done!
            {/if}
        </div>
    </div>
</DropReceiver>

<style>
    :global(#remux-container) {
        display: flex;
        justify-content: center;
        align-items: center;
        width: 100%;
    }

    #remux-open {
        display: flex;
        flex-direction: row;
        justify-content: center;
        align-items: center;
        text-align: center;
        gap: 48px;
        transition:
            transform 0.2s,
            opacity 0.2s;
    }

    #remux-processing {
        position: absolute;
        display: flex;
        flex-direction: column;
        opacity: 0;
        transform: scale(0.9);
        transition:
            transform 0.2s,
            opacity 0.2s;
        pointer-events: none;
    }

    #remux-processing.processing {
        opacity: 1;
        transform: none;
    }

    #remux-open.processing {
        transform: scale(0.9);
        opacity: 0;
        pointer-events: none;
    }

    #processing-status {
        display: flex;
        flex-direction: column;
        padding: var(--padding);
        gap: var(--padding);
        justify-content: center;
    }

    .progress-bar {
        height: 20px;
        width: 400px;
        max-width: 400px;
        border-radius: 6px;
        background: var(--button);
    }

    .progress-text {
        font-size: 14px;
        text-align: center;
    }

    #remux-receiver {
        max-width: 450px;
    }

    #remux-bullets {
        display: flex;
        flex-direction: column;
        gap: 18px;
        max-width: 450px;
    }

    @media screen and (max-width: 920px) {
        #remux-open {
            flex-direction: column;
            gap: var(--padding);
        }

        #remux-bullets {
            padding: var(--padding);
        }
    }

    @media screen and (max-width: 535px) {
        .progress-bar {
            width: 350px;
        }

        #remux-bullets {
            gap: var(--padding);
        }
    }

    @media screen and (max-height: 750px) and (max-width: 535px) {
        :global(#remux-container:not(.processing)) {
            justify-content: start;
            align-items: start;
            padding-top: var(--padding);
        }
    }
</style>
