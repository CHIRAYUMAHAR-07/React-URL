# React-URL
React URL Web App

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Affordmed Link Manager</title>
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" />
    <link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons" />
    <script src="https://cdn.jsdelivr.net/npm/react@18.2.0/umd/react.development.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/react-dom@18.2.0/umd/react-dom.development.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mui/material@5.13.5/umd/material-ui.production.min.js"></script>
</head>
<body>
    <div id="root"></div>
    <script>
        const { useState, useEffect } = React;
        const { Container, TextField, Button, Dialog, DialogTitle, DialogContent, DialogActions, Typography, Alert, Box, Table, TableBody, TableCell, TableContainer, TableHead, TableRow, Paper } = MaterialUI;

        const recordLog = (note) => {
            const logEntries = JSON.parse(localStorage.getItem('logs')) || [];
            logEntries.push({ note, time: new Date().toISOString() });
            localStorage.setItem('logs', JSON.stringify(logEntries));
        };

        function LinkApp() {
            const [inputList, setInputList] = useState([{ url: '', code: '', expiry: '' }]);
            const [outputList, setOutputList] = useState([]);
            const [notification, setNotification] = useState(null);
            const [showStats, setShowStats] = useState(false);
            const [storedLinks, updateStoredLinks] = useState(() => {
                try {
                    return JSON.parse(localStorage.getItem('shortLinks')) || {};
                } catch (e) {
                    recordLog('Unable to read local storage');
                    return {};
                }
            });

            useEffect(() => {
                if (window.location.host !== 'localhost:3000') {
                    setNotification('This app is designed to run on http://localhost:3000');
                }

                const currentPath = window.location.pathname.substring(1);
                if (currentPath === 'stats') return setShowStats(true);

                const matched = storedLinks[currentPath];
                if (matched) {
                    const now = new Date();
                    if (new Date(matched.expiry) > now) {
                        matched.clicks = matched.clicks || [];
                        matched.clicks.push({ time: new Date().toISOString(), ref: document.referrer || 'Direct', region: 'Unknown' });
                        storedLinks[currentPath] = matched;
                        localStorage.setItem('shortLinks', JSON.stringify(storedLinks));
                        recordLog(Redirecting: ${matched.url});
                        window.location.href = matched.url;
                    } else {
                        setNotification('Link expired.');
                    }
                }
            }, []);

            const generateCode = () => {
                let token;
                do {
                    token = Array.from({ length: 6 }, () => Math.random().toString(36).charAt(2)).join('');
                } while (storedLinks[token]);
                return token;
            };

            const validateCode = (txt) => /^[a-zA-Z0-9]{1,10}$/.test(txt);
            const validateURL = (txt) => /^https?:\/\/.+\..+/.test(txt);

            const modifyInput = (idx, key, val) => {
                const copied = [...inputList];
                copied[idx][key] = val;
                setInputList(copied);
            };

            const addNewInput = () => {
                if (inputList.length < 5) {
                    setInputList([...inputList, { url: '', code: '', expiry: '' }]);
                }
            };

            const processAll = (e) => {
                e.preventDefault();
                const logs = [];
                const updated = { ...storedLinks };

                for (const { url, code, expiry } of inputList) {
                    if (!validateURL(url)) return setNotification('One or more URLs are invalid.');

                    const finalCode = code ? (validateCode(code) ? code : null) : generateCode();
                    if (!finalCode || updated[finalCode]) return setNotification('Invalid or duplicate code.');

                    const expiration = new Date(Date.now() + ((expiry ? parseInt(expiry) : 30) * 60000));
                    updated[finalCode] = { url, code: finalCode, expiry: expiration.toISOString(), clicks: [], created: new Date().toISOString() };
                    logs.push({ short: http://localhost:3000/${finalCode}, original: url, expires: expiration.toLocaleString() });
                    recordLog(Created: ${finalCode});
                }

                updateStoredLinks(updated);
                localStorage.setItem('shortLinks', JSON.stringify(updated));
                setOutputList(logs);
                setNotification(null);
            };

            if (showStats) {
                return (
                    <Container maxWidth="md" sx={{ mt: 5 }}>
                        <Typography variant="h4" gutterBottom>Affordmed – Link Insights</Typography>
                        <TableContainer component={Paper}>
                            <Table>
                                <TableHead>
                                    <TableRow>
                                        <TableCell>Shortened</TableCell>
                                        <TableCell>Target URL</TableCell>
                                        <TableCell>Created At</TableCell>
                                        <TableCell>Expires</TableCell>
                                        <TableCell>Hits</TableCell>
                                    </TableRow>
                                </TableHead>
                                <TableBody>
                                    {Object.entries(storedLinks).map(([code, data]) => (
                                        <TableRow key={code}>
                                            <TableCell>{http://localhost:3000/${code}}</TableCell>
                                            <TableCell>{data.url}</TableCell>
                                            <TableCell>{new Date(data.created).toLocaleString()}</TableCell>
                                            <TableCell>{new Date(data.expiry).toLocaleString()}</TableCell>
                                            <TableCell>{data.clicks?.length || 0}</TableCell>
                                        </TableRow>
                                    ))}
                                </TableBody>
                            </Table>
                        </TableContainer>
                    </Container>
                );
            }

            return (
                <Container maxWidth="md" sx={{ mt: 5 }}>
                    <Typography variant="h4" align="center" gutterBottom>Affordmed Link Manager</Typography>
                    <form onSubmit={processAll}>
                        {inputList.map((item, idx) => (
                            <Box key={idx} mb={2}>
                                <TextField label="Original URL" fullWidth margin="dense" value={item.url} onChange={(e) => modifyInput(idx, 'url', e.target.value)} />
                                <TextField label="Preferred Shortcode" fullWidth margin="dense" value={item.code} onChange={(e) => modifyInput(idx, 'code', e.target.value)} />
                                <TextField label="Validity (minutes)" type="number" fullWidth margin="dense" value={item.expiry} onChange={(e) => modifyInput(idx, 'expiry', e.target.value)} />
                            </Box>
                        ))}
                        <Box mt={2}>
                            <Button variant="outlined" onClick={addNewInput} disabled={inputList.length >= 5}>Add Another</Button>
                        </Box>
                        <Box mt={2}>
                            <Button variant="contained" color="primary" type="submit">Generate Links</Button>
                        </Box>
                        {notification && <Alert severity="error" sx={{ mt: 2 }}>{notification}</Alert>}
                    </form>
                    <Box mt={4}>
                        {outputList.map((entry, idx) => (
                            <Alert key={idx} severity="success" sx={{ mb: 2 }}>
                                <strong>{entry.original}</strong> ➝ <a href={entry.short} target="_blank">{entry.short}</a> (expires: {entry.expires})
                            </Alert>
                        ))}
                    </Box>
                </Container>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<LinkApp />);
    </script>
</body>
</html>



