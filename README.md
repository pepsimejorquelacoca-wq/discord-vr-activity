/**
 * @name PlatformSpoof
 * @author k1ng_op
 * @description Spoof your Discord client platform status
 * @version 1.0.1
 * @authorId 641266820187160576
 * @authorLink https://github.com/k1ng0p
 * @source https://github.com/k1ng0p/PlatformSpoof
 */

const IDENTIFY = 2;
const BD = new BdApi("PlatformSpoof");
const { React } = BdApi;

const PLATFORMS = [
    { value: "off",     label: "Off",                          props: null },
    { value: "desktop", label: "Desktop (Windows)",             props: { os: "Windows",     browser: "Discord Client",   device: "" } },
    { value: "web",     label: "Web / Browser (Chrome)",        props: { os: "Linux",       browser: "Chrome",           device: "" } },
    { value: "mobile",  label: "Mobile (Discord Android)",      props: { os: "Android",     browser: "Discord Android",  device: "Discord Android" } },
    { value: "meta",    label: "Meta Quest / VR → VR: Online",  props: { os: "Android",     browser: "Discord VR",       device: "Meta Quest" } },
    { value: "console", label: "Console",                       props: { os: "Playstation", browser: "Discord Embedded", device: "PlayStation" } },
];

function getSpoofProps(platform) {
    return PLATFORMS.find(p => p.value === platform)?.props ?? null;
}

module.exports = class PlatformSpoof {
    constructor() {
        this._origSend = null;
        this._socket = null;
    }

    getSetting() {
        return BD.Data.load("platform") ?? "mobile";
    }

    setSetting(val) {
        BD.Data.save("platform", val);
    }

    getSocket() {
        return BdApi.Webpack.getByKeys("getSocket", "isConnected")?.getSocket() ?? null;
    }

    patchSocket(socket) {
        if (!socket || socket.__psPatched) return;
        this._origSend = socket.send.bind(socket);
        const self = this;
        socket.send = function(op, data, flag) {
            if (op === IDENTIFY && data?.properties) {
                const spoof = getSpoofProps(self.getSetting());
                if (spoof) Object.assign(data.properties, spoof);
            }
            return self._origSend.call(this, op, data, flag);
        };
        socket.__psPatched = true;
        this._socket = socket;
    }

    unpatchSocket() {
        if (this._socket && this._origSend) {
            this._socket.send = this._origSend;
            delete this._socket.__psPatched;
        }
        this._origSend = null;
        this._socket = null;
    }

    forceIdentify() {
        if (this.getSetting() === "off") return;

        const socket = this.getSocket();
        if (!socket) return;

        if (!socket.__psPatched) {
            this.unpatchSocket();
            this.patchSocket(socket);
        }

        socket.sessionId = null;
        socket.seq = 0;

        const ws = socket.webSocket;
        if (ws && ws.readyState !== WebSocket.CLOSED && ws.readyState !== WebSocket.CLOSING) {
            ws.close(1000);
        } else if (!ws) {
            socket.close();
            setTimeout(() => socket.connect(), 500);
        }
    }

    start() {
        const socket = this.getSocket();
        if (!socket) return BdApi.Logger.error("PlatformSpoof", "socket not found");

        this.patchSocket(socket);

        const GCS = BdApi.Webpack.getByKeys("getSocket", "isConnected");
        if (GCS?.isConnected()) this.forceIdentify();

        window.__ps = {
            reconnect: () => this.forceIdentify(),
            sessions:  () => {
                const s = BdApi.Webpack.getByKeys("getSessions")?.getSessions?.();
                if (!s) return;
                Object.values(s).forEach(x =>
                    console.log(x.sessionId?.slice(0, 8), "→", x.clientInfo?.client, "/", x.clientInfo?.os)
                );
            },
            status: () => {
                const s = this.getSocket();
                console.log("patched:", !!s?.__psPatched, "| platform:", this.getSetting(), "| session:", s?.sessionId, "| state:", s?.connectionState);
            },
        };
    }

    stop() {
        this.unpatchSocket();
        delete window.__ps;
    }

    // stateful settings panel - fixes dropdown not updating / reconnect not firing
    getSettingsPanel() {
        const { SettingItem, DropdownInput } = BdApi.Components;
        if (!SettingItem || !DropdownInput) {
            const div = document.createElement("div");
            div.textContent = "PlatformSpoof: your BD version is missing required UI components.";
            div.style.cssText = "padding:10px;color:var(--text-danger)";
            return div;
        }
        return React.createElement(this._Panel(SettingItem, DropdownInput));
    }

    _Panel(SettingItem, DropdownInput) {
        const self = this;
        return function Panel() {
            const [value, setValue] = React.useState(self.getSetting());

            const onChange = (val) => {
                self.setSetting(val);
                setValue(val);
                self.forceIdentify();
            };

            return React.createElement(SettingItem, {
                id: "ps-platform",
                name: "Platform",
                note: "Changes apply immediately via gateway reconnect.",
            },
                React.createElement(DropdownInput, {
                    options: PLATFORMS.map(p => ({ label: p.label, value: p.value })),
                    value: value,
                    onChange: onChange,
                })
            );
        };
    }
};
