// server.js
const express = require('express');
const app = express();
const http = require('http').Server(app);
const io = require('socket.io')(http);

app.use(express.json());

let auctions = []; // Array per gestire le aste

// Endpoint per creare un'asta
app.post('/create-auction', (req, res) => {
    const auction = {
        id: auctions.length + 1,
        product: req.body.product,
        endTime: Date.now() + 8000, // 8 secondi
        bids: []
    };
    auctions.push(auction);
    res.status(201).send(auction);
});

// Endpoint per fare un'offerta
app.post('/bid', (req, res) => {
    const auction = auctions.find(a => a.id === req.body.auctionId);
    if (auction) {
        auction.bids.push({ user: req.body.user, amount: req.body.amount });
        io.emit('new-bid', auction);
        res.status(200).send(auction);
    } else {
        res.status(404).send({ message: 'Auction not found' });
    }
});

// Timer per gestire le aste
setInterval(() => {
    const now = Date.now();
    auctions.forEach(auction => {
        if (auction.endTime <= now) {
            io.emit('auction-ended', auction);
            auctions = auctions.filter(a => a.id !== auction.id);
        }
    });
}, 1000);

http.listen(3000, () => {
    console.log('Server is running on port 3000');
});// App.js
import React, { useState, useEffect } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:3000');

function App() {
    const [auctions, setAuctions] = useState([]);

    useEffect(() => {
        socket.on('new-bid', (auction) => {
            setAuctions(prevAuctions => prevAuctions.map(a => a.id === auction.id ? auction : a));
        });

        socket.on('auction-ended', (auction) => {
            setAuctions(prevAuctions => prevAuctions.filter(a => a.id !== auction.id));
        });
    }, []);

    return (
        <div>
            <h1>Astevip</h1>
            {auctions.map(auction => (
                <div key={auction.id}>
                    <h2>{auction.product}</h2>
                    <p>Ends in: {Math.max(0, (auction.endTime - Date.now()) / 1000)} seconds</p>
                    <ul>
                        {auction.bids.map((bid, index) => (
                            <li key={index}>{bid.user}: ${bid.amount}</li>
                        ))}
                    </ul>
                </div>
            ))}
        </div>
    );
}

export default App;// auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const router = express.Router();

const users = []; // Array per memorizzare gli utenti

// Registrazione
router.post('/register', async (req, res) => {
    const hashedPassword = await bcrypt.hash(req.body.password, 10);
    const user = { id: users.length + 1, username: req.body.username, password: hashedPassword };
    users.push(user);
    res.status(201).send({ message: 'User registered' });
});

// Login
router.post('/login', async (req, res) => {
    const user = users.find(u => u.username === req.body.username);
    if (user && await bcrypt.compare(req.body.password, user.password)) {
        const token = jwt.sign({ id: user.id }, 'secret_key', { expiresIn: '1h' });
        res.status(200).send({ token });
    } else {
        res.status(401).send({ message: 'Invalid credentials' });
    }
});

// Middleware per autenticazione
function authenticateToken(req, res, next) {
    const token = req.headers['authorization'];
    if (!token) return res.status(401).send({ message: 'Access denied' });

    jwt.verify(token, 'secret_key', (err, user) => {
        if (err) return res.status(403).send({ message: 'Invalid token' });
        req.user = user;
        next();
    });
}

module.exports = { router, authenticateToken };// server.js
const express = require('express');
const http = require('http').Server(app);
const io = require('socket.io')(http);
const { router: authRouter, authenticateToken } = require('./auth');

const app = express();
app.use(express.json());
app.use('/auth', authRouter);

let auctions = [];

// Endpoint per creare un'asta (protetto)
app.post('/create-auction', authenticateToken, (req, res) => {
    const auction = {
        id: auctions.length + 1,
        product: req.body.product,
        endTime: Date.now() + 8000,
        bids: []
    };
    auctions.push(auction);
    res.status(201).send(auction);
});

// Endpoint per fare un'offerta (protetto)
app.post('/bid', authenticateToken, (req, res) => {
    const auction = auctions.find(a => a.id === req.body.auctionId);
    if (auction) {
        auction.bids.push({ user: req.user.id, amount: req.body.amount });
        io.emit('new-bid', auction);
        res.status(200).send(auction);
    } else {
        res.status(404).send({ message: 'Auction not found' });
    }
});

// Timer per gestire le aste
setInterval(() => {
    const now = Date.now();
    auctions.forEach(auction => {
        if (auction.endTime <= now) {
            io.emit('auction-ended', auction);
            auctions = auctions.filter(a => a.id !== auction.id);
        }
    });
}, 1000);

http.listen(3000, () => {
    console.log('Server is running on port 3000');// payment.js
const express = require('express');
const stripe = require('stripe')('your_stripe_secret_key');
const router = express.Router();

router.post('/create-payment-intent', async (req, res) => {
    const paymentIntent = await stripe.paymentIntents.create({
        amount: req.body.amount,
        currency: 'usd',
    });
    res.send({
        clientSecret: paymentIntent.client_secret,
    });
});

module.exports = router;
});// server.js
const paymentRouter = require('./payment');
app.use('/payment', paymentRouter);// App.js
import React, { useState, useEffect } from 'react';
import io from 'socket.io-client';
import axios from 'axios';
import { loadStripe } from '@stripe/stripe-js';
import { Elements } from '@stripe/react-stripe-js';
import CheckoutForm from './CheckoutForm';

const socket = io('http://localhost:3000');
const stripePromise = loadStripe('your_stripe_public_key');

function App() {
    const [auctions, setAuctions] = useState([]);
    const [token, setToken] = useState(null);

    useEffect(() => {
        socket.on('new-bid', (auction) => {
            setAuctions(prevAuctions => prevAuctions.map(a => a.id === auction.id ? auction : a));
        });

        socket.on('auction-ended', (auction) => {
            setAuctions(prevAuctions => prevAuctions.filter(a => a.id !== auction.id));
        });
    }, []);

    const handleLogin = async (username, password) => {
        const response = await axios.post('http://localhost:3000/auth/login', { username, password });
        setToken(response.data.token);
    };

    const handleBid = async (auctionId, amount) => {
        await axios.post('http://localhost:3000/bid', { auctionId, amount }, {
            headers: { Authorization: token }
        });
    };

    return (
        <div>
            <h1>Astevip</h1>
            <LoginForm onLogin={handleLogin} />
            {auctions.map(auction => (
                <div key={auction.id}>
                    <h2>{auction.product}</h2>
                    <p>Ends in: {Math.max(0, (auction.endTime - Date.now()) / 1000)} seconds</p>
                    <ul>
                        {auction.bids.map((bid, index) => (
                            <li key={index}>{bid.user}: ${bid.amount}</li>
                        ))}
                    </ul>
                    <button onClick={() => handleBid(auction.id, 100)}>Bid $100</button>
                </div>
            ))}
            <Elements stripe={stripePromise}>
                <CheckoutForm />
            </Elements>
        </div>
    );
}

function LoginForm({ onLogin }) {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        onLogin(username, password);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input type="text" value={username} onChange={(e) => setUsername(e.target.value)} placeholder="Username" />
            <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
            <button type="submit">Login</button>
        </form>
    );
}

export default App;// CheckoutForm.js
import React from 'react';
import { CardElement, useStripe, useElements } from '@stripe/react-stripe-js';
import axios from 'axios';

function CheckoutForm() {
    const stripe = useStripe();
    const elements = useElements();

    const handleSubmit = async (event) => {
        event.preventDefault();

        const { error, paymentMethod } = await stripe.createPaymentMethod({
            type: 'card',
            card: elements.getElement(CardElement),
        });

        if (!error) {
            const { id } = paymentMethod;
            const response = await axios.post('http://localhost:3000/payment/create-payment-intent', {
                amount: 1000, // Amount in cents
                payment_method: id,
            });

            const { clientSecret } = response.data;
            const confirmPayment = await stripe.confirmCardPayment(clientSecret);

            if (confirmPayment.error) {
                console.error(confirmPayment.error.message);
            } else {
                console.log('Payment successful');
            }
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <CardElement />
            <button type="submit" disabled={!stripe}>Pay</button>
        </form>
    );
}

export default CheckoutForm;
