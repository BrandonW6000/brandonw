# brandonw
A ultraviolet render deploy
//Be careful that some websites will noy be avaliable on ultraviolet, error on line 452
import type { LookupOneOptions } from 'node:dns';
import EventEmitter from 'node:events';
import { readFileSync } from 'node:fs';
import type {
	Agent as HttpAgent,
	IncomingMessage,
	ServerResponse,
} from 'node:http';
import type { Agent as HttpsAgent } from 'node:https';
import { join } from 'node:path';
import { Readable, type Duplex } from 'node:stream';
import type { ReadableStream } from 'node:stream/web';
import createHttpError from 'http-errors';
import type WebSocket from 'ws';
import type { JSONDatabaseAdapter } from './Meta.js';
import { nullMethod } from './requestUtil.js';

export interface BareRequest extends Request {
	native: IncomingMessage;
}

export interface BareErrorBody {
	code: string;
	id: string;
	message?: string;
	stack?: string;
}

export class BareError extends Error {
	status: number;
	body: BareErrorBody;
	constructor(status: number, body: BareErrorBody) {
		super(body.message || body.code);
		this.status = status;
		this.body = body;
	}
}

export const pkg = JSON.parse(
	readFileSync(join(__dirname, '..', 'package.json'), 'utf-8')
) as { version: string };

const project: BareProject = {
	name: 'bare-server-node',
	description: 'TOMPHTTP NodeJS Bare Server',
	repository: 'https://github.com/tomphttp/bare-server-node',
	version: pkg.version,
};

export function json<T>(status: number, json: T) {
	const send = Buffer.from(JSON.stringify(json, null, '\t'));

	return new Response(send, {
		status,
		headers: {
			'content-type': 'application/json',
			'content-length': send.byteLength.toString(),
		},
	});
}

export type BareMaintainer = {
	email?: string;
	website?: string;
};

export type BareProject = {
	name?: string;
	description?: string;
	email?: string;
	website?: string;
	repository?: string;
	version?: string;
};

export type BareLanguage =
	| 'NodeJS'
	| 'ServiceWorker'
	| 'Deno'
	| 'Java'
	| 'PHP'
	| 'Rust'
	| 'C'
	| 'C++'
	| 'C#'
	| 'Ruby'
	| 'Go'
	| 'Crystal'
	| 'Shell'
	| string;

export type BareManifest = {
	maintainer?: BareMaintainer;
	project?: BareProject;
	versions: string[];
	language: BareLanguage;
	memoryUsage?: number;
};

export interface Options {
	logErrors: boolean;
	/**
	 * Callback for filtering the remote URL.
	 * @returns Nothing
	 * @throws An error if the remote is bad.
	 */
	filterRemote?: (remote: Readonly<URL>) => Promise<void> | void;
	/**
	 * DNS lookup
	 * May not get called when remote.host is an IP
	 * Use in combination with filterRemote to block IPs
	 */
	lookup: (
		hostname: string,
		options: LookupOneOptions,
		callback: (
			err: NodeJS.ErrnoException | null,
			address: string,
			family: number
		) => void
	) => void;
	localAddress?: string;
	family?: number;
	maintainer?: BareMaintainer;
	httpAgent: HttpAgent;
	httpsAgent: HttpsAgent;
	database: JSONDatabaseAdapter;
	wss: WebSocket.Server;
}

export type RouteCallback = (
	request: BareRequest,
	response: ServerResponse<IncomingMessage>,
	options: Options
) => Promise<Response> | Response;

export type SocketRouteCallback = (
	request: BareRequest,
	socket: Duplex,
	head: Buffer,
	options: Options
) => Promise<void> | void;

export default class Server extends EventEmitter {
	directory: string;
	routes = new Map<string, RouteCallback>();
	socketRoutes = new Map<string, SocketRouteCallback>();
	versions: string[] = [];
	private closed = false;
	private options: Options;
	/**
	 * @internal
	 */
	constructor(directory: string, options: Options) {
		super();
		this.directory = directory;
		this.options = options;
	}
	/**
	 * Remove all timers and listeners
	 */
	close() {
		this.closed = true;
		this.emit('close');
	}
	shouldRoute(request: IncomingMessage): boolean {
		return (
			!this.closed &&
			request.url !== undefined &&
			request.url.startsWith(this.directory)
		);
	}
	get instanceInfo(): BareManifest {
		return {
			versions: this.versions,
			language: 'NodeJS',
			memoryUsage:
				Math.round((process.memoryUsage().heapUsed / 1024 / 1024) * 100) / 100,
			maintainer: this.options.maintainer,
			project,
		};
	}
	async routeUpgrade(req: IncomingMessage, socket: Duplex, head: Buffer) {
		const request = new Request(new URL(req.url!, 'http://bare-server-node'), {
			method: req.method,
			body: nullMethod.includes(req.method || '') ? undefined : req,
			headers: req.headers as HeadersInit,
		}) as BareRequest;

		request.native = req;

		const service = new URL(request.url).pathname.slice(
			this.directory.length - 1
		);

		if (this.socketRoutes.has(service)) {
			const call = this.socketRoutes.get(service)!;

			try {
				await call(request, socket, head, this.options);
			} catch (error) {
				if (this.options.logErrors) {
					console.error(error);
				}

				socket.end();
			}
		} else {
			socket.end();
		}
	}
	async routeRequest(req: IncomingMessage, res: ServerResponse) {
		const request = new Request(new URL(req.url!, 'http://bare-server-node'), {
			method: req.method,
			body: nullMethod.includes(req.method || '') ? undefined : req,
			headers: req.headers as HeadersInit,
		}) as BareRequest;

		request.native = req;

		const service = new URL(request.url).pathname.slice(
			this.directory.length - 1
		);
		let response: Response;

		try {
			if (request.method === 'OPTIONS') {
				response = new Response(undefined, { status: 200 });
			} else if (service === '/') {
				response = json(200, this.instanceInfo);
			} else if (this.routes.has(service)) {
				const call = this.routes.get(service)!;
				response = await call(request, res, this.options);
			} else {
				throw new createHttpError.NotFound();
			}
		} catch (error) {
			if (this.options.logErrors) console.error(error);

			if (createHttpError.isHttpError(error)) {
				response = json(error.statusCode, {
					code: 'UNKNOWN',
					id: `error.${error.name}`,
					message: error.message,
					stack: error.stack,
				});
			} else if (error instanceof Error) {
				response = json(500, {
					code: 'UNKNOWN',
					id: `error.${error.name}`,
					message: error.message,
					stack: error.stack,
				});
			} else {
				response = json(500, {
					code: 'UNKNOWN',
					id: 'error.Exception',
					message: error,
					stack: new Error(<string | undefined>error).stack,
				});
			}

			if (!(response instanceof Response)) {
				if (this.options.logErrors) {
					console.error(
						'Cannot',
						request.method,
						new URL(request.url).pathname,
						': Route did not return a response.'
					);
				}

				throw new createHttpError.InternalServerError();
			}
		}

		response.headers.set('x-robots-tag', 'noindex');
		response.headers.set('access-control-allow-headers', '*');
		response.headers.set('access-control-allow-origin', '*');
		response.headers.set('access-control-allow-methods', '*');
		response.headers.set('access-control-expose-headers', '*');
		// don't fetch preflight on every request...
		// instead, fetch preflight every 10 minutes
		response.headers.set('access-control-max-age', '7200');

		res.writeHead(
			response.status,
			response.statusText,
			Object.fromEntries(response.headers)
		);

		if (response.body) {
			const body = Readable.fromWeb(response.body as ReadableStream);
			body.pipe(res);
			res.on('close', () => body.destroy());
		} else res.end();
	}
}
import http from 'node:http';
import { createBareServer } from '@tomphttp/bare-server-node';
import { SocksProxyAgent } from 'socks-proxy-agent';

const socksProxyAgent = new SocksProxyAgent(
	'socks://your-name@gmail.com:abcdef12345124@br41.nordvpn.com'
);

const httpServer = http.createServer();

const bareServer = createBareServer('/', {
	httpAgent: socksProxyAgent,
	httpsAgent: socksProxyAgent,
});

httpServer.on('request', (req, res) => {
	if (bareServer.shouldRoute(req)) {
		bareServer.routeRequest(req, res);
	} else {
		res.writeHead(400);
		res.end('Not found.');
	}
});

httpServer.on('upgrade', (req, socket, head) => {
	if (bareServer.shouldRoute(req)) {
		bareServer.routeUpgrade(req, socket, head);
	} else {
		socket.end();
	}
});

httpServer.on('listening', () => {
	console.log('HTTP server listening');
});

httpServer.listen({
	port: 8080,
});
import type { BareHeaders } from './requestUtil.js';

export function objectFromRawHeaders(raw: string[]): BareHeaders {
	const result: BareHeaders = Object.create(null);

	for (let i = 0; i < raw.length; i += 2) {
		const [header, value] = raw.slice(i, i + 2);
		if (header in result) {
			const v = result[header];
			if (Array.isArray(v)) v.push(value);
			else result[header] = [v, value];
		} else result[header] = value;
	}

	return result;
}

export function rawHeaderNames(raw: string[]) {
	const result: string[] = [];

	for (let i = 0; i < raw.length; i += 2) {
		if (!result.includes(raw[i])) result.push(raw[i]);
	}

	return result;
}

export function mapHeadersFromArray(from: string[], to: BareHeaders) {
	for (const header of from) {
		if (header.toLowerCase() in to) {
			const value = to[header.toLowerCase()];
			delete to[header.toLowerCase()];
			to[header] = value;
		}
	}

	return to;
}

/**
 * Converts a header into an HTTP-ready comma joined header.
 */
export function flattenHeader(value: string | string[]) {
	return Array.isArray(value) ? value.join(', ') : value;
}
{
	"name": "@tomphttp/bare-server-node",
	"description": "The Bare Server implementation in NodeJS.",
	"version": "2.0.1",
	"homepage": "https://github.com/tomphttp",
	"bugs": {
		"url": "https://github.com/tomphttp/bare-server-node/issues",
		"email": "tomp@sys32.dev"
	},
	"repository": {
		"type": "git",
		"url": "https://github.com/tomphttp/bare-server-node.git"
	},
	"author": {
		"name": "TOMP Development",
		"email": "tomp@sys32.dev",
		"url": "https://github.com/tomphttp"
	},
	"keywords": [
		"proxy",
		"tomp",
		"tomphttp"
	],
	"license": "GPL-3.0",
	"type": "commonjs",
	"engines": {
		"node": ">=18.0.0"
	},
	"bin": {
		"bare-server-node": "bin.js"
	},
	"main": "dist/createServer.js",
	"types": "dist/createServer.d.ts",
	"files": [
		"dist",
		"bin.js"
	],
	"scripts": {
		"build": "tsc",
		"dev": "tsc --watch"
	},
	"dependencies": {
		"async-exit-hook": "^2.0.1",
		"commander": "^10.0.1",
		"dotenv": "^16.0.3",
		"http-errors": "^2.0.0",
		"ipaddr.js": "^2.0.1",
		"source-map-support": "^0.5.21",
		"ws": "^8.13.0"
	},
	"devDependencies": {
		"@ianvs/prettier-plugin-sort-imports": "^4.0.0",
		"@types/async-exit-hook": "^2.0.0",
		"@types/http-errors": "^2.0.1",
		"@types/node": "^18.16.19",
		"@types/source-map-support": "^0.5.6",
		"@types/ws": "^8.5.4",
		"@typescript-eslint/eslint-plugin": "^5.59.7",
		"@typescript-eslint/parser": "^5.59.7",
		"eslint": "^8.41.0",
		"prettier": "^2.8.8",
		"typescript": "^5.0.4",
		"undici": "^5.22.1"
	}
}
import { createBareServer } from '@tomphttp/bare-server-node';
```
