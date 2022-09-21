## Goal

Allow features to refer to an identifier that is unique to the dev container they are installed into and that is stable across rebuilds.

E.g., the `docker-in-docker` feature needs a way to mount a volume per dev container.

## Proposal

The identifier will be referred to as `${devcontainerId}` in the feature metadata and that will be replaced with the dev container's id. It should only be used in parts of the metadata that is not used for building the image because that would otherwise prevent pre-building the image at a time when the dev container's id is not known yet. Excluding boolean and enum properties the properties supporting `${devcontainerId}` are: `entrypoint`, `mounts`, `customizations`.

Implementations can choose how to compute this identifier. They must ensure that it is unique among other dev containers on the same Docker host and that it is stable across rebuilds of dev containers. The identifier must only contain alphanumeric characters. We describe a way to do this below.

### Label-based Implementation 

The following assumes that a dev container can be identified among other dev containers on the same Docker host by a set of labels on the container. Implementations may choose to follow this approach.

The identifier is derived from the set of container labels uniquely identifying the dev container. It is up to the implementation to choose these labels. E.g., if the dev container is based on a local folder the label could be named `devcontainer.local_folder` and have the local folder's path as its value.

E.g., the `docker-in-docker` feature could use the dev container id with:

```jsonc
{
    "id": "docker-in-docker",
    "version": "1.0.4",
    // ...
    "mounts": [
        {
            "source": "dind-var-lib-docker-${devcontainerId}",
            "target": "/var/lib/docker",
            "type": "volume"
        }
    ]
}
```

### Label-based Computation

- Input the labels as a JSON object with the object's keys being the label names and the object's values being the labels' values.
	- To ensure implementations get to the same result, the object keys must be sorted and any optional whitespace outside of the keys and values must be removed.
- Compute a SHA-256 hash from the UTF-8 encoded input string.
- Use a base-32 encoded representation left-padded with '0' to 52 characters as the result.

JavaScript implementation taking an object with the labels as argument and returning a string as the result:
```js
const crypto = require('crypto');

function uniqueIdForLabels(idLabels) {
	const stringInput = JSON.stringify(idLabels, Object.keys(idLabels).sort()); // sort properties
	const bufferInput = Buffer.from(stringInput, 'utf-8');

	const hash = crypto.createHash('sha256')
		.update(bufferInput)
		.digest();

	const uniqueId = BigInt(`0x${hash.toString('hex')}`)
		.toString(32)
		.padStart(52, '0');
	return uniqueId;
}
```