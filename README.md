# __

<p>Click <em>Sample</em>, and you'll see the output is very random, because the
neural network hasn't trained yet. Now click <em>Run</em> to start training (<a
href="https://raw.githubusercontent.com/karpathy/makemore/master/names.txt">data</a>).
When the loss is under 2.6, click <em>Sample</em> again, and you'll see the output
has gotten a bit better. Try to reach a loss of 2.5 or lower. Note that the neural network tries to predict only based on
the previous character, so it won't get very good!</p>
<p>This neural network is written in JS with zero dependencies (apart from the plot library).</p>
<input type="text" id="learningRateInput" value="10">
<button id="runButton">Run</button>
<code>loss: </code><samp id="lossOutput">x.xxxx</samp>;
<code>iterations: </code><samp id="iterationsOutput">x</samp><hr>
<button id="sampleButton">Sample</button>
<samp id="sampleOutput"></samp>
<hr>
<p>Each cell contains the "count" for the bigram. The weights of the network is a 27x27 matrix that contains the logits (or log counts). The table contains the exponentiated weights. The weights are initalised randomly.</p>
<div style="float: left;" id="table"></div>
<div style="float: left;" id="losses"></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/plotly.js/2.26.0/plotly.min.js"></script>
<script type="module">
    const response = await fetch('https://raw.githubusercontent.com/karpathy/makemore/master/names.txt');
    const text = await response.text();
    const names = text.split('\n');
    const indexToCharMap = [ '.', ...new Set( names.join('') ) ].sort();
    const stringToCharMap = {};

    for ( let i = indexToCharMap.length; i--; ) {
        stringToCharMap[ indexToCharMap[ i ] ] = i;
    }

    console.log( indexToCharMap, stringToCharMap );

    let xs = []; // Inputs.
    let ys = []; // Targets, or labels.

    for ( const name of names ) {
        const exploded = '.' + name + '.';
        let i = 1;
        while ( exploded[ i ] ) {
            xs.push( stringToCharMap[ exploded[ i - 1 ] ] );
            ys.push( stringToCharMap[ exploded[ i ] ] );
            i++;
        }
    }

    function clone( array, shape = array.shape ) {
        const clone = new Float32Array( array );
        clone.shape = shape;
        return clone;
    }

    function empty( shape ) {
        const array = new Float32Array( shape.reduce( ( a, b ) => a * b, 1 ) );
        array.shape = shape;
        return array;
    }

    function oneHot( a, length ) {
        const B = empty( [ a.length, length ] );
        for ( let i = a.length; i--; ) B[ i * length + a[ i ] ] = 1;
        return B;
    }

    const totalChars = indexToCharMap.length;
    const XOneHot = oneHot( xs, totalChars );
    const W = empty( [ totalChars, totalChars ] );
    for ( let i = W.length; i--; ) W[ i ] = Math.random() * 2 - 1;
    const losses = [];
    let learningRate;

    async function iteration() {
        const Wx = await matMul( XOneHot, W );
        const probs = softmaxByRow( Wx );
        const [m, n] = probs.shape;

        let sum = 0;
        for ( let m_ = m; m_--; ) {
            // Sum the logProbs (log likelihoods) of the correct label.
            sum += Math.log( probs[ m_ * n + ys[ m_ ] ] );
        }

        const mean = sum / m;
        // Mean negative log likelihood.
        const loss = - mean;

        losses.push( loss );

        // Backpropagation.
        const WxGradient = clone( probs );

        for ( let m_ = m; m_--; ) {
            // Subtract 1 for the gradient of the correct label.
            WxGradient[ m_ * n + ys[ m_ ] ] -= 1;
            for ( let n_ = n; n_--; ) {
                // Divide by the number of rows.
                WxGradient[ m_ * n + n_ ] /= m;
            }
        }

        const WGradient = await matMul( transpose( XOneHot ), WxGradient );

        for ( let i = W.length; i--; ) W[ i ] -= learningRate * WGradient[ i ];
    }

    function matMul(A, B) {
        const [ m, n ] = A.shape;
        const [ p, q ] = B.shape;
        const C = empty( [ m, q ] );

        if ( n !== p ) {
            throw new Error( 'Matrix dimensions do not match.' );
        }

        for ( let m_ = m; m_--; ) {
            for ( let q_ = q; q_--; ) {
                let sum = 0;
                for ( let n_ = n; n_--; ) {
                    sum += A[m_ * n + n_] * B[n_ * q + q_];
                }
                C[m_ * q + q_] = sum;
            }
        }

        return C;
    }

    function softmaxByRow( A ) {
        const [m, n] = A.shape;
        const B = empty(A.shape);
        for ( let m_ = m; m_--; ) {
            let max = -Infinity;
            for ( let n_ = n; n_--; ) {
                const value = A[m_ * n + n_];
                if (value > max) max = value;
            }
            let sum = 0;
            for ( let n_ = n; n_--; ) {
                const i = m_ * n + n_;
                // Subtract the max to avoid overflow
                sum += B[i] = Math.exp(A[i] - max);
            }
            for ( let n_ = n; n_--; ) {
                B[m_ * n + n_] /= sum;
            }
        }
        return B;
    }

    function transpose( A ) {
        const [ m, n ] = A.shape;
        const B = empty( [ n, m ] );

        for ( let m_ = m; m_--; ) {
            for ( let n_ = n; n_--; ) {
                B[n_ * m + m_] = A[m_ * n + n_];
            }
        }

        return B;
    }

    async function sampleNames() {
        const names = [];

        for (let i = 0; i < 5; i++) {
            const indices = [ 0 ];

            do {
                const context = indices.slice( -1 );
                const Wc = await matMul( oneHot( context, totalChars ), W );
                const probs = softmaxByRow( Wc );
                indices.push( sample( probs ) );
            } while ( indices[ indices.length - 1 ] );

            const name = indices.slice( 1, -1 ).map( ( i ) => indexToCharMap[ i ] ).join( '' );
            names.push( name );
        }

        return names;
    }

    function sample(probs) {
        const sample = Math.random();
        let total = 0;
        for ( let i = probs.length; i--; ) {
            total += probs[ i ];
            if ( sample < total ) return i;
        }
    }

    let running = false;

    function setRunning( value ) {
        running = value;
        runButton.textContent = running ? 'Stop' : 'Run';
        learningRateInput.disabled = running;
        sampleButton.disabled = running;
    }

    async function run() {
        if ( running ) {
            await iteration();
            updateUI();
            requestAnimationFrame( run );
        }
    }

    runButton.onclick = () => {
        if ( running ) {
            setRunning( false );
        } else {
            learningRate = parseFloat( learningRateInput.value );
            setRunning( true );
            run();
        }
    }

    sampleButton.onclick = async () => {
        sampleOutput.innerText = ( await sampleNames() ).join( ', ' );
    }

    createHeatMap();
    createLossesGraph();

    function createHeatMap() {
        const counts = clone( W );

        for ( let i = counts.length; i--; ) {
            counts[ i ] = Math.exp( counts[ i ] );
        }

        function flatTo2D(A) {
            const [rows, cols] = A.shape;
            const result = [];

            for (let i = 0; i < rows; i++) {
                const row = [];
                for (let j = 0; j < cols; j++) {
                    row.push(A[i * cols + j]);
                }
                result.push(row);
            }

            return result;
        }

        const annotations = [];
        for(let i = 0; i < indexToCharMap.length; i++) {
            for(let j = 0; j < indexToCharMap.length; j++) {
                annotations.push({
                    x: indexToCharMap[j],
                    y: indexToCharMap[i],
                    text: `${indexToCharMap[i]}${indexToCharMap[j]}`,
                    showarrow: false,
                    font: { color: 'white' }
                });
            }
        }

        Plotly.react('table', [
            {
                x: indexToCharMap,
                y: indexToCharMap,
                z: flatTo2D( counts ),
                type: 'heatmap',
                colorscale: [ [ 0, 'white' ], [ 1, 'black' ] ],
                showscale: false,
            },
        ], {
            width: 600,
            height: 600,
            yaxis: {
                autorange: 'reversed',
                tickvals: [],
            },
            xaxis: {
                tickvals: [],
            },
            margin: { t: 10, b: 10, l: 10, r: 10 },
            annotations,
        });
    }

    function createLossesGraph() {
        Plotly.react('losses', [
            {
                x: Array.from( { length: losses.length }, ( _, i ) => i ),
                y: losses,
                name: 'Losses',
                hoverinfo: 'none'
            },
        ], {
            title: 'Losses',
            width: 600,
            height: 600,
            yaxis: {
                title: 'Loss',
                type: 'log'
            },
            xaxis: {
                title: 'Iterations'
            }
        });
    }

    let totalIterations = 0;

    function updateUI() {
        lossOutput.innerText = losses[ losses.length - 1 ].toFixed( 4 );
        totalIterations++;
        iterationsOutput.innerText = totalIterations;
        createHeatMap();
        createLossesGraph();
    }

    if ( ! navigator.gpu ) {
        alert( 'This browser does not support WebGPU. Falling back to CPU.' );
    }
</script>
