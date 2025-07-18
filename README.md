<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>PVA PRODUCCIÓN</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Carga de librerías de React -->
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <!-- Carga de Babel para transpilar JSX en el navegador -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Carga de librerías adicionales -->
    <script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
</head>
<body class="bg-gray-100">
    <div id="root"></div>

    <script type="text/babel">
        // =================================================================
        // INICIO DEL CÓDIGO DE LA APLICACIÓN
        // =================================================================

        const { useState, useCallback, useEffect, useMemo, useRef } = React;

        // --- 1. CONFIGURACIÓN Y TIPOS ---

        // ¡IMPORTANTE! Reemplaza esta URL con la tuya.
        const GOOGLE_SHEET_API_URL = "https://script.google.com/macros/s/AKfycbySIfnRbrTnAuidskAMBOhQL0q5Sef9NpvcTonFX9kDyedhn41CIQCqNcWZ2ortl8Br2w/exec"; 

        const TERMINALS = [
            { name: "Menú", path: "/" },
            { name: "Órdenes", path: "/admin" },
            { name: "Materia Prima", path: "/raw-material" },
            { name: "Transformación", path: "/transformation" },
            { name: "Bodega", path: "/warehouse" },
            { name: "Materiales", path: "/materials" },
            { name: "Informes", path: "/reports" },
        ];

        const ProductionOrderStatus = {
            PENDING: 'PENDIENTE',
            IN_PROGRESS: 'EN PRODUCCIÓN',
            COMPLETED: 'COMPLETADO',
            IN_WAREHOUSE: 'EN BODEGA',
        };

        const MaterialType = {
            RAW: 'Materia Prima',
            FINISHED: 'Producto Terminado',
            BYPRODUCT: 'Subproducto',
            PACKAGING: 'Material de Empaque',
        };

        const UnitOfMeasure = {
            KG: 'kilos',
            UNIT: 'unidad',
            PACK: 'paquete',
            BUNDLE: 'bulto',
            BOX: 'caja',
            LITER: 'litros',
        };

        // --- 2. COMPONENTES DE LA INTERFAZ (UI) ---

        const Modal = ({ isOpen, onClose, title, children }) => {
            if (!isOpen) return null;
            return (
                <div className="fixed inset-0 bg-black bg-opacity-50 z-50 flex justify-center items-center p-4" aria-modal="true" role="dialog">
                    <div className="bg-white rounded-lg shadow-xl w-full max-w-lg">
                        <div className="p-4 border-b flex justify-between items-center">
                            <h3 className="text-lg font-semibold">{title}</h3>
                            <button onClick={onClose} className="text-gray-500 hover:text-gray-800" aria-label="Cerrar modal">&times;</button>
                        </div>
                        <div className="p-6 overflow-y-auto max-h-[80vh]">{children}</div>
                    </div>
                </div>
            );
        };

        const Card = ({ children }) => <div className="bg-white rounded-lg shadow-md">{children}</div>;
        const CardHeader = ({ children }) => <div className="p-4 border-b">{children}</div>;
        const CardContent = ({ children }) => <div className="p-4">{children}</div>;
        const CardFooter = ({ children }) => <div className="p-4 border-t bg-gray-50">{children}</div>;
        
        // --- 3. PANTALLAS DE LA APLICACIÓN (PAGES) ---

        const AdminTerminal = ({ orders, createOrder, createMultipleOrders, deleteOrder, materials, isSaving }) => {
            const [isOrderModalOpen, setIsOrderModalOpen] = useState(false);
            const [newOrder, setNewOrder] = useState({ productReference: '', desiredQuantity: '', deliveryDate: '' });
            const [isBatchModalOpen, setIsBatchModalOpen] = useState(false);
            const batchFileInputRef = useRef(null);

            const finishedProducts = useMemo(() => 
                materials.filter(m => m.type === MaterialType.FINISHED), 
                [materials]
            );

            const handleNewOrderChange = (e) => {
                const { name, value } = e.target;
                setNewOrder(prev => ({ ...prev, [name]: value }));
            };

            const handleNewOrderSubmit = (e) => {
                e.preventDefault();
                if (!newOrder.productReference || !newOrder.desiredQuantity || !newOrder.deliveryDate) {
                    alert("Por favor, completa todos los campos.");
                    return;
                }
                createOrder(newOrder.productReference, parseFloat(newOrder.desiredQuantity), newOrder.deliveryDate);
                setIsOrderModalOpen(false);
                setNewOrder({ productReference: '', desiredQuantity: '', deliveryDate: '' });
            };

            const handleBatchFileImport = (event) => {
                const file = event.target.files[0];
                if (file) {
                    Papa.parse(file, {
                        header: true,
                        skipEmptyLines: true,
                        complete: (results) => {
                            const newOrders = results.data.map(row => ({
                                productReference: row.productReference,
                                desiredQuantity: parseFloat(row.desiredQuantity),
                                deliveryDate: row.deliveryDate
                            })).filter(o => o.productReference && !isNaN(o.desiredQuantity) && o.deliveryDate);
                            createMultipleOrders(newOrders);
                            setIsBatchModalOpen(false);
                        }
                    });
                }
            };
            
            return (
                <div>
                    <div className="mb-4 flex justify-end gap-2">
                        <button onClick={() => setIsOrderModalOpen(true)} className="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg flex items-center gap-2">Crear Orden</button>
                        <button onClick={() => setIsBatchModalOpen(true)} className="bg-green-600 text-white font-bold py-2 px-4 rounded-lg flex items-center gap-2">Importar CSV</button>
                    </div>
                    <Card>
                        <CardHeader><h2 className="font-bold">Órdenes de Producción</h2></CardHeader>
                        <CardContent>
                            <div className="overflow-x-auto">
                                <table className="min-w-full">
                                    <thead>
                                        <tr className="bg-gray-100">
                                            <th className="p-2 text-left">ID</th>
                                            <th className="p-2 text-left">Referencia</th>
                                            <th className="p-2 text-left">Cantidad</th>
                                            <th className="p-2 text-left">Fecha Entrega</th>
                                            <th className="p-2 text-left">Estado</th>
                                            <th className="p-2"></th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        {orders.map(order => (
                                            <tr key={order.id} className="border-b">
                                                <td className="p-2">{order.id}</td>
                                                <td className="p-2">{order.productReference}</td>
                                                <td className="p-2">{order.desiredQuantity}</td>
                                                <td className="p-2">{new Date(order.deliveryDate).toLocaleDateString('es-ES', { timeZone: 'UTC' })}</td>
                                                <td className="p-2"><span className="px-2 py-1 text-xs font-semibold rounded-full bg-yellow-200 text-yellow-800">{order.status}</span></td>
                                                <td className="p-2"><button onClick={() => deleteOrder(order.id)} className="text-red-500" aria-label={`Eliminar orden ${order.id}`}>Eliminar</button></td>
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        </CardContent>
                    </Card>

                    <Modal isOpen={isOrderModalOpen} onClose={() => setIsOrderModalOpen(false)} title="Crear Nueva Orden de Producción">
                        <form onSubmit={handleNewOrderSubmit} className="space-y-4">
                            <label className="block">
                                <span className="text-gray-700">Referencia del Producto:</span>
                                <select
                                    name="productReference"
                                    value={newOrder.productReference}
                                    onChange={handleNewOrderChange}
                                    className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
                                    required
                                >
                                    <option value="">Selecciona un producto terminado</option>
                                    {finishedProducts.map(product => (
                                        <option key={product.id} value={product.materialName}>{product.materialName}</option>
                                    ))}
                                </select>
                            </label>
                            <label className="block">
                                <span className="text-gray-700">Cantidad Deseada:</span>
                                <input
                                    type="number"
                                    name="desiredQuantity"
                                    value={newOrder.desiredQuantity}
                                    onChange={handleNewOrderChange}
                                    className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
                                    required
                                />
                            </label>
                            <label className="block">
                                <span className="text-gray-700">Fecha de Entrega:</span>
                                <input
                                    type="date"
                                    name="deliveryDate"
                                    value={newOrder.deliveryDate}
                                    onChange={handleNewOrderChange}
                                    className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
                                    required
                                />
                            </label>
                            <button type="submit" className="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg hover:bg-blue-700 disabled:bg-blue-400" disabled={isSaving}>
                                {isSaving ? 'Guardando...' : 'Crear Orden'}
                            </button>
                        </form>
                    </Modal>

                    <Modal isOpen={isBatchModalOpen} onClose={() => setIsBatchModalOpen(false)} title="Crear Órdenes por Lotes (CSV)">
                        <div className="space-y-4">
                            <p className="text-gray-700">Sube un archivo CSV con las columnas: `productReference`, `desiredQuantity`, `deliveryDate`.</p>
                            <input
                                type="file"
                                accept=".csv"
                                ref={batchFileInputRef}
                                onChange={handleBatchFileImport}
                                className="block w-full text-sm text-gray-500
                                    file:mr-4 file:py-2 file:px-4
                                    file:rounded-full file:border-0
                                    file:text-sm file:font-semibold
                                    file:bg-blue-50 file:text-blue-700
                                    hover:file:bg-blue-100"
                            />
                        </div>
                    </Modal>
                </div>
            );
        };

        const RawMaterialTerminal = ({ orders, assignRawMaterial, materials, isSaving }) => {
            const [isAssignModalOpen, setIsAssignModalOpen] = useState(false);
            const [currentOrderToAssign, setCurrentOrderToAssign] = useState(null);
            const [selectedMaterials, setSelectedMaterials] = useState([]);

            const rawMaterials = useMemo(() => 
                materials.filter(m => m.type === MaterialType.RAW), 
                [materials]
            );

            const openAssignModal = (order) => {
                setCurrentOrderToAssign(order);
                setSelectedMaterials([]);
                setIsAssignModalOpen(true);
            };

            const handleMaterialSelectionChange = (materialId, quantity) => {
                setSelectedMaterials(prev => {
                    const existing = prev.find(item => item.materialId === materialId);
                    const numQuantity = parseFloat(quantity);
                    if (existing) {
                        if (numQuantity > 0) {
                            return prev.map(item => 
                                item.materialId === materialId ? { ...item, quantity: numQuantity } : item
                            );
                        } else {
                            return prev.filter(item => item.materialId !== materialId);
                        }
                    } else if (numQuantity > 0) {
                        return [...prev, { materialId, quantity: numQuantity }];
                    }
                    return prev;
                });
            };

            const handleAssignSubmit = (e) => {
                e.preventDefault();
                if (currentOrderToAssign && selectedMaterials.length > 0) {
                    assignRawMaterial(currentOrderToAssign.id, selectedMaterials);
                    setIsAssignModalOpen(false);
                    setCurrentOrderToAssign(null);
                } else {
                    alert("Por favor, selecciona al menos una materia prima y especifica una cantidad.");
                }
            };

            return (
                <div>
                    <Card>
                        <CardHeader><h2 className="font-bold">Asignación de Materia Prima</h2></CardHeader>
                        <CardContent>
                            <p>Aquí se mostrarán las órdenes **PENDIENTES** para asignación de materia prima.</p>
                            {orders.length === 0 ? (
                                <p className="text-gray-500 mt-4">No hay órdenes pendientes de materia prima.</p>
                            ) : (
                                <div className="overflow-x-auto">
                                    <table className="min-w-full divide-y divide-gray-200">
                                        <thead className="bg-gray-50">
                                            <tr className="bg-gray-100">
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">ID Orden</th>
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Producto</th>
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Cantidad</th>
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Fecha Entrega</th>
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Estado</th>
                                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                                            </tr>
                                        </thead>
                                        <tbody className="bg-white divide-y divide-gray-200">
                                            {orders.map(order => (
                                                <tr key={order.id}>
                                                    <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{order.id}</td>
                                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{order.productReference}</td>
                                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{order.desiredQuantity}</td>
                                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{new Date(order.deliveryDate).toLocaleDateString('es-ES', { timeZone: 'UTC' })}</td>
                                                    <td className="px-3 py-2 whitespace-nowrap text-sm">
                                                        <span className="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-yellow-100 text-yellow-800">
                                                            {order.status}
                                                        </span>
                                                    </td>
                                                    <td className="px-3 py-2 whitespace-nowrap text-left text-sm font-medium">
                                                        <button 
                                                            onClick={() => openAssignModal(order)} 
                                                            className="bg-indigo-500 text-white text-xs px-3 py-1 rounded-md hover:bg-indigo-600 disabled:bg-indigo-300"
                                                            disabled={isSaving}
                                                        >
                                                            Asignar
                                                        </button>
                                                    </td>
                                                </tr>
                                            ))}
                                        </tbody>
                                    </table>
                                </div>
                            )}
                        </CardContent>
                    </Card>

                    <Modal 
                        isOpen={isAssignModalOpen} 
                        onClose={() => setIsAssignModalOpen(false)} 
                        title={`Asignar Materia Prima a Orden: ${currentOrderToAssign?.id || ''}`}
                    >
                        <form onSubmit={handleAssignSubmit} className="space-y-4">
                            <p className="text-gray-700 mb-4">Selecciona las materias primas y las cantidades a asignar:</p>
                            {rawMaterials.length === 0 ? (
                                <p className="text-red-500">No hay materias primas definidas. Por favor, ve a "Materiales" para definirlas.</p>
                            ) : (
                                <div className="grid grid-cols-1 gap-4 max-h-60 overflow-y-auto pr-2">
                                    {rawMaterials.map(material => {
                                        const assignedQuantity = selectedMaterials.find(sm => sm.materialId === material.id)?.quantity || '';
                                        return (
                                            <div key={material.id} className="flex items-center gap-4 p-2 border rounded-md bg-gray-50">
                                                <label htmlFor={`material-${material.id}`} className="flex-grow text-gray-700 font-medium">
                                                    {material.materialName} ({material.unit})
                                                </label>
                                                <input
                                                    id={`material-${material.id}`}
                                                    type="number"
                                                    min="0"
                                                    step="any"
                                                    value={assignedQuantity}
                                                    onChange={(e) => handleMaterialSelectionChange(material.id, e.target.value)}
                                                    className="w-24 px-2 py-1 border rounded-md text-sm"
                                                    placeholder="Cantidad"
                                                />
                                            </div>
                                        );
                                    })}
                                </div>
                            )}
                            <div className="flex justify-end pt-4 border-t mt-4">
                                <button type="submit" className="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg hover:bg-blue-700 disabled:bg-blue-400" disabled={isSaving}>
                                    {isSaving ? 'Asignando...' : 'Confirmar Asignación'}
                                </button>
                            </div>
                        </form>
                    </Modal>
                </div>
            );
        };
        
        const MaterialsTerminal = ({ materials, createMaterial, createMultipleMaterials, deleteMaterial, isSaving }) => {
            const [isMaterialModalOpen, setIsMaterialModalOpen] = useState(false);
            const [newMaterial, setNewMaterial] = useState({ 
                materialCode: '', 
                materialName: '', 
                unit: '', 
                type: '', 
                recipe: '' 
            });
            const [isXLSXModalOpen, setIsXLSXModalOpen] = useState(false);
            const xlsxFileInputRef = useRef(null);

            const handleNewMaterialChange = (e) => {
                const { name, value } = e.target;
                setNewMaterial(prev => ({ ...prev, [name]: value }));
            };

            const handleNewMaterialSubmit = (e) => {
                e.preventDefault();
                createMaterial(newMaterial);
                setIsMaterialModalOpen(false);
                setNewMaterial({ materialCode: '', materialName: '', unit: '', type: '', recipe: '' });
            };

            const handleXLSXFileImport = (event) => {
                const file = event.target.files[0];
                if (file) {
                    const reader = new FileReader();
                    reader.onload = (e) => {
                        const data = new Uint8Array(e.target.result);
                        const workbook = XLSX.read(data, { type: 'array' });
                        const sheetName = workbook.SheetNames[0];
                        const worksheet = workbook.Sheets[sheetName];
                        const json = XLSX.utils.sheet_to_json(worksheet, { header: 1 });

                        const headers = json[0];
                        const rows = json.slice(1);

                        const newMaterials = rows.map(row => {
                            const material = {};
                            headers.forEach((header, index) => {
                                let key = '';
                                if (header === 'Codigo de material') key = 'materialCode';
                                else if (header === 'nombre de material') key = 'materialName';
                                else if (header === 'unidad de medida') key = 'unit';
                                else if (header === 'tipo') key = 'type';
                                else if (header === 'receta') key = 'recipe';
                                
                                if (key) {
                                    material[key] = row[index];
                                }
                            });
                            return material;
                        }).filter(m => m.materialCode && m.materialName && m.unit && m.type);

                        createMultipleMaterials(newMaterials);
                        setIsXLSXModalOpen(false);
                    };
                    reader.readAsArrayBuffer(file);
                }
            };

            const downloadXLSXTemplate = () => {
                const ws_data = [
                    ["Codigo de material", "nombre de material", "unidad de medida", "tipo", "receta"],
                    ["MP-001", "Azúcar", "kilos", "Materia Prima", ""],
                    ["PT-001", "Pastel de Chocolate", "unidad", "Producto Terminado", "Azúcar (1kg), Harina (2kg)"],
                    ["EMP-001", "Caja de Cartón", "unidad", "Material de Empaque", ""],
                ];
                const ws = XLSX.utils.aoa_to_sheet(ws_data);
                const wb = XLSX.utils.book_new();
                XLSX.utils.book_append_sheet(wb, ws, "Plantilla Materiales");
                XLSX.writeFile(wb, "Plantilla_Materiales_PVA.xlsx");
            };
            
            return (
                <div>
                    <div className="mb-4 flex justify-end gap-2">
                        <button onClick={() => setIsMaterialModalOpen(true)} className="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg flex items-center gap-2">Crear Material</button>
                        <button onClick={() => setIsXLSXModalOpen(true)} className="bg-green-600 text-white font-bold py-2 px-4 rounded-lg flex items-center gap-2">Importar XLSX</button>
                    </div>
                    <Card>
                        <CardHeader><h2 className="font-bold">Definición de Materiales</h2></CardHeader>
                        <CardContent>
                            <div className="overflow-x-auto">
                                <table className="min-w-full divide-y divide-gray-200">
                                    <thead className="bg-gray-50">
                                        <tr className="bg-gray-100">
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Código</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nombre</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Unidad</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Tipo</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Receta</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                                        </tr>
                                    </thead>
                                    <tbody className="bg-white divide-y divide-gray-200">
                                        {materials.map(material => (
                                            <tr key={material.id}>
                                                <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{material.materialCode}</td>
                                                <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{material.materialName}</td>
                                                <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{material.unit}</td>
                                                <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{material.type}</td>
                                                <td className="px-3 py-2 whitespace-normal text-sm text-gray-500">{material.recipe || 'N/A'}</td>
                                                <td className="px-3 py-2 whitespace-nowrap text-right text-sm font-medium">
                                                    <button onClick={() => deleteMaterial(material.id)} className="text-red-600 hover:text-red-900" aria-label={`Eliminar material ${material.materialName}`}>Eliminar</button>
                                                </td>
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        </CardContent>
                    </Card>

                    <Modal isOpen={isMaterialModalOpen} onClose={() => setIsMaterialModalOpen(false)} title="Crear Nuevo Material">
                        <form onSubmit={handleNewMaterialSubmit} className="space-y-4">
                            <label className="block">
                                <span className="text-gray-700">Código de Material:</span>
                                <input type="text" name="materialCode" value={newMaterial.materialCode} onChange={handleNewMaterialChange} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required />
                            </label>
                            <label className="block">
                                <span className="text-gray-700">Nombre de Material:</span>
                                <input type="text" name="materialName" value={newMaterial.materialName} onChange={handleNewMaterialChange} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required />
                            </label>
                            <label className="block">
                                <span className="text-gray-700">Unidad de Medida:</span>
                                <select name="unit" value={newMaterial.unit} onChange={handleNewMaterialChange} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required>
                                    <option value="">Selecciona una unidad</option>
                                    {Object.values(UnitOfMeasure).map(unit => <option key={unit} value={unit}>{unit}</option>)}
                                </select>
                            </label>
                            <label className="block">
                                <span className="text-gray-700">Tipo:</span>
                                <select name="type" value={newMaterial.type} onChange={handleNewMaterialChange} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required>
                                    <option value="">Selecciona un tipo</option>
                                    {Object.values(MaterialType).map(type => <option key={type} value={type}>{type}</option>)}
                                </select>
                            </label>
                            {newMaterial.type === MaterialType.FINISHED && (
                                <label className="block">
                                    <span className="text-gray-700">Receta (ej. Harina (1kg), Azúcar (0.5kg)):</span>
                                    <textarea name="recipe" value={newMaterial.recipe} onChange={handleNewMaterialChange} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" rows="3"></textarea>
                                </label>
                            )}
                            <button type="submit" className="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg hover:bg-blue-700 disabled:bg-blue-400" disabled={isSaving}>
                                {isSaving ? 'Guardando...' : 'Crear Material'}
                            </button>
                        </form>
                    </Modal>

                    <Modal isOpen={isXLSXModalOpen} onClose={() => setIsXLSXModalOpen(false)} title="Importar Materiales (XLSX)">
                        <div className="space-y-4">
                            <p className="text-gray-700">Sube un archivo XLSX con las columnas: `Codigo de material`, `nombre de material`, `unidad de medida`, `tipo`, `receta`.</p>
                            <button onClick={downloadXLSXTemplate} className="bg-blue-500 text-white font-bold py-2 px-4 rounded-lg hover:bg-blue-600">Descargar Plantilla</button>
                            <input
                                type="file"
                                accept=".xlsx, .xls"
                                ref={xlsxFileInputRef}
                                onChange={handleXLSXFileImport}
                                className="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-green-50 file:text-green-700 hover:file:bg-green-100"
                            />
                        </div>
                    </Modal>
                </div>
            );
        };
        
        const ReportsPage = ({ orders, materials }) => {
            const exportToExcel = () => {
                const dataToExport = orders.map(order => ({
                    'ID Orden': order.id,
                    'Referencia Producto': order.productReference,
                    'Cantidad Deseada': order.desiredQuantity,
                    'Fecha Creación': new Date(order.creationDate).toLocaleDateString('es-ES', { timeZone: 'UTC' }),
                    'Fecha Entrega': new Date(order.deliveryDate).toLocaleDateString('es-ES', { timeZone: 'UTC' }),
                    'Estado': order.status,
                    'Materia Prima Asignada': (order.assignedRawMaterials || []).map(m => {
                        const materialDef = materials.find(def => def.id === m.materialId);
                        return materialDef ? `${materialDef.materialName} (${m.quantity} ${materialDef.unit})` : `ID: ${m.materialId} (${m.quantity})`;
                    }).join(', '),
                    'Productos Terminados': (order.finishedProducts || []).map(p => `${p.reference} (${p.quantity})`).join(', '),
                    'Subproductos Generados': (order.generatedByProducts || []).map(b => `${b.reference} (${b.quantity})`).join(', '),
                    'Ubicación Bodega': order.warehouseLocation || 'N/A',
                }));

                const ws = XLSX.utils.json_to_sheet(dataToExport);
                const wb = XLSX.utils.book_new();
                XLSX.utils.book_append_sheet(wb, ws, "Órdenes de Producción");
                XLSX.writeFile(wb, "PVA_Produccion_Reporte.xlsx");
            };

            return (
                <Card>
                    <CardHeader><h2 className="font-bold">Informes de Producción</h2></CardHeader>
                    <CardContent>
                        <p className="mb-4 text-gray-700">Aquí puedes ver un resumen de todas las órdenes de producción y exportarlas.</p>
                        <button onClick={exportToExcel} className="bg-teal-600 text-white font-bold py-2 px-4 rounded-lg hover:bg-teal-700 mb-4">Exportar a Excel</button>
                        <div className="overflow-x-auto">
                            <table className="min-w-full divide-y divide-gray-200">
                                <thead className="bg-gray-50">
                                    <tr className="bg-gray-100">
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">ID</th>
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Referencia</th>
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Estado</th>
                                    </tr>
                                </thead>
                                <tbody className="bg-white divide-y divide-gray-200">
                                    {orders.map((order) => (
                                        <tr key={order.id}>
                                            <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{order.id}</td>
                                            <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{order.productReference}</td>
                                            <td className="px-3 py-2 whitespace-nowrap text-sm">{order.status}</td>
                                        </tr>
                                    ))}
                                </tbody>
                            </table>
                        </div>
                    </CardContent>
                </Card>
            );
        };
        
        const HomePage = ({ onNavigate }) => {
            const menuItems = [
                { name: "Gestión de Órdenes", terminal: "Órdenes", color: "bg-sky-600" },
                { name: "Materia Prima", terminal: "Materia Prima", color: "bg-rose-600" },
                { name: "Transformación", terminal: "Transformación", color: "bg-purple-600" },
                { name: "Bodega", terminal: "Bodega", color: "bg-teal-600" },
                { name: "Materiales", terminal: "Materiales", color: "bg-orange-600" },
                { name: "Informes", terminal: "Informes", color: "bg-amber-500" },
            ];
            return (
                <div className="flex flex-col items-center justify-center p-4">
                    <header className="text-center mb-10">
                        <h1 className="text-4xl font-bold text-slate-800">PVA PRODUCCIÓN</h1>
                        <p className="text-lg text-gray-500 mt-2">Sistema de Gestión y Trazabilidad</p>
                    </header>
                    <main className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6 w-full max-w-4xl">
                        {menuItems.map((item) => (
                            <button
                                key={item.terminal}
                                onClick={() => onNavigate(item.terminal)}
                                className={`group p-8 rounded-2xl shadow-lg text-white text-center font-semibold text-xl ${item.color} hover:opacity-90 transition-opacity duration-200`}
                            >
                                {item.name}
                            </button>
                        ))}
                    </main>
                </div>
            );
        };
        
        const TransformationTerminal = ({ orders, materials, recordTransformationDetails, isSaving }) => {
            const [isRecordTransformationModalOpen, setIsRecordTransformationModalOpen] = useState(false);
            const [currentOrderToTransform, setCurrentOrderToTransform] = useState(null);
            const [transformationDetails, setTransformationDetails] = useState({ finishedProducts: [], byProducts: [] });

            const ordersInProgress = orders.filter(o => o.status === ProductionOrderStatus.IN_PROGRESS);

            const openRecordTransformationModal = (order) => {
                setCurrentOrderToTransform(order);
                const initialFinished = order.productReference ? [{ reference: order.productReference, quantity: order.desiredQuantity }] : [];
                setTransformationDetails({ finishedProducts: initialFinished, byProducts: [] });
                setIsRecordTransformationModalOpen(true);
            };

            const handleTransformationChange = (type, index, field, value) => {
                setTransformationDetails(prev => {
                    const newList = [...prev[type]];
                    newList[index][field] = value;
                    return { ...prev, [type]: newList };
                });
            };
            
            const addTransformationItem = (type) => {
                setTransformationDetails(prev => ({
                    ...prev,
                    [type]: [...prev[type], { reference: '', quantity: '' }]
                }));
            };

            const removeTransformationItem = (type, index) => {
                setTransformationDetails(prev => ({
                    ...prev,
                    [type]: prev[type].filter((_, i) => i !== index)
                }));
            };

            const handleRecordTransformationSubmit = (e) => {
                e.preventDefault();
                recordTransformationDetails(currentOrderToTransform.id, transformationDetails);
                setIsRecordTransformationModalOpen(false);
            };

            return (
                <div>
                    <Card>
                        <CardHeader><h2 className="font-bold">Terminal de Transformación</h2></CardHeader>
                        <CardContent>
                            <p className="text-gray-700 mb-2">Órdenes <span className="font-bold text-blue-600">EN PRODUCCIÓN</span> listas para registrar detalles de transformación.</p>
                            <div className="overflow-x-auto">
                                <table className="min-w-full divide-y divide-gray-200">
                                    <thead className="bg-gray-50">
                                        <tr>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">ID Orden</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Producto</th>
                                            <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Acción</th>
                                        </tr>
                                    </thead>
                                    <tbody className="bg-white divide-y divide-gray-200">
                                    {ordersInProgress.map(order => (
                                        <tr key={order.id}>
                                            <td className="px-3 py-2 whitespace-nowrap">{order.id}</td>
                                            <td className="px-3 py-2 whitespace-nowrap">{order.productReference}</td>
                                            <td className="px-3 py-2 whitespace-nowrap">
                                                <button onClick={() => openRecordTransformationModal(order)} className="bg-green-500 text-white px-3 py-1 text-sm rounded-md hover:bg-green-600">Registrar</button>
                                            </td>
                                        </tr>
                                    ))}
                                    </tbody>
                                </table>
                            </div>
                            {ordersInProgress.length === 0 && <p className="text-gray-500 mt-4">No hay órdenes en producción.</p>}
                        </CardContent>
                    </Card>

                    <Modal isOpen={isRecordTransformationModalOpen} onClose={() => setIsRecordTransformationModalOpen(false)} title={`Registrar Transformación para Orden: ${currentOrderToTransform?.id}`}>
                        <form onSubmit={handleRecordTransformationSubmit} className="space-y-6">
                            <div>
                                <h4 className="font-semibold text-gray-800 mb-2">Productos Terminados</h4>
                                {transformationDetails.finishedProducts.map((p, index) => (
                                    <div key={index} className="flex gap-2 items-center mb-2 p-2 bg-gray-50 rounded-md">
                                        <input type="text" placeholder="Referencia" value={p.reference} onChange={e => handleTransformationChange('finishedProducts', index, 'reference', e.target.value)} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required/>
                                        <input type="number" placeholder="Cantidad" value={p.quantity} onChange={e => handleTransformationChange('finishedProducts', index, 'quantity', parseFloat(e.target.value))} className="mt-1 block w-24 rounded-md border-gray-300 shadow-sm" required/>
                                        <button type="button" onClick={() => removeTransformationItem('finishedProducts', index)} className="text-red-500 hover:bg-red-100 rounded-full p-1">&times;</button>
                                    </div>
                                ))}
                                <button type="button" onClick={() => addTransformationItem('finishedProducts')} className="text-sm text-blue-600 mt-1 hover:underline">Añadir producto</button>
                            </div>
                             <div>
                                <h4 className="font-semibold text-gray-800 mb-2">Subproductos Generados</h4>
                                {transformationDetails.byProducts.map((p, index) => (
                                    <div key={index} className="flex gap-2 items-center mb-2 p-2 bg-gray-50 rounded-md">
                                        <input type="text" placeholder="Referencia" value={p.reference} onChange={e => handleTransformationChange('byProducts', index, 'reference', e.target.value)} className="mt-1 block w-full rounded-md border-gray-300 shadow-sm" required/>
                                        <input type="number" placeholder="Cantidad" value={p.quantity} onChange={e => handleTransformationChange('byProducts', index, 'quantity', parseFloat(e.target.value))} className="mt-1 block w-24 rounded-md border-gray-300 shadow-sm" required/>
                                         <button type="button" onClick={() => removeTransformationItem('byProducts', index)} className="text-red-500 hover:bg-red-100 rounded-full p-1">&times;</button>
                                    </div>
                                ))}
                                <button type="button" onClick={() => addTransformationItem('byProducts')} className="text-sm text-blue-600 mt-1 hover:underline">Añadir subproducto</button>
                            </div>
                            <div className="pt-4 border-t">
                                <button type="submit" className="bg-green-600 text-white font-bold py-2 px-4 rounded-lg w-full hover:bg-green-700 disabled:bg-green-400" disabled={isSaving}>
                                    {isSaving ? 'Guardando...' : 'Confirmar Transformación'}
                                </button>
                            </div>
                        </form>
                    </Modal>
                </div>
            );
        };
        
        const WarehouseTerminal = ({ orders, confirmWarehouseReceipt, isSaving }) => {
            const ordersToReceive = orders.filter(o => o.status === ProductionOrderStatus.COMPLETED);

            return (
                <Card>
                    <CardHeader><h2 className="font-bold">Terminal de Bodega</h2></CardHeader>
                    <CardContent>
                         <p className="text-gray-700 mb-2">Órdenes <span className="font-bold text-green-600">COMPLETADAS</span> listas para recepción en bodega.</p>
                         <div className="overflow-x-auto">
                            <table className="min-w-full divide-y divide-gray-200">
                                <thead className="bg-gray-50">
                                    <tr>
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">ID Orden</th>
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Detalles</th>
                                        <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">Acción</th>
                                    </tr>
                                </thead>
                                <tbody className="bg-white divide-y divide-gray-200">
                                {ordersToReceive.map(order => (
                                    <tr key={order.id}>
                                        <td className="px-3 py-2 whitespace-nowrap font-medium">{order.id}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm">
                                            <p className="font-semibold">{order.productReference}</p>
                                            {(order.finishedProducts || []).length > 0 && <p>PT: {(order.finishedProducts || []).map(p => `${p.reference} (${p.quantity})`).join(', ')}</p>}
                                            {(order.generatedByProducts || []).length > 0 && <p>SP: {(order.generatedByProducts || []).map(p => `${p.reference} (${p.quantity})`).join(', ')}</p>}
                                        </td>
                                        <td className="px-3 py-2 whitespace-nowrap">
                                            <button onClick={() => confirmWarehouseReceipt(order.id)} className="bg-teal-500 text-white px-3 py-1 rounded text-sm hover:bg-teal-600 disabled:bg-teal-300" disabled={isSaving}>Recibir en Bodega</button>
                                        </td>
                                    </tr>
                                ))}
                                </tbody>
                            </table>
                        </div>
                        {ordersToReceive.length === 0 && <p className="text-gray-500 mt-4">No hay órdenes completadas para recibir.</p>}
                    </CardContent>
                </Card>
            );
        };

        // --- 4. COMPONENTE PRINCIPAL (App) ---
        
        const App = () => {
            const [appData, setAppData] = useState({ productionOrders: [], materialDefinitions: [] });
            const [isLoading, setIsLoading] = useState(true);
            const [isSaving, setIsSaving] = useState(false);
            const [error, setError] = useState(null);
            const [activeTerminal, setActiveTerminal] = useState('Menú');

            const fetchData = useCallback(async () => {
                if (GOOGLE_SHEET_API_URL.includes("AKfycbwivsMJmLVLGU_guBom95qLmkbPOkPMSS3h_Lp8Vl95CgavRiR5UmYk3fjxFJhGrOyhVA")) {
                    setError("Error de Configuración: La URL de la API es la de ejemplo. Reemplázala con tu URL de Google Apps Script para empezar a usar la aplicación.");
                    setIsLoading(false);
                    return;
                }
                setIsLoading(true);
                setError(null);
                try {
                    const response = await fetch(`${GOOGLE_SHEET_API_URL}?action=getData`);
                    const data = await response.json();
                    if (data.status === 'SUCCESS') {
                        setAppData({
                            productionOrders: data.orders || [],
                            materialDefinitions: data.materials || [],
                        });
                    } else {
                        throw new Error(data.message || 'Error desconocido del API');
                    }
                } catch (err) {
                    setError('Fallo al cargar los datos: ' + err.message);
                } finally {
                    setIsLoading(false);
                }
            }, []);
            
            const saveData = useCallback(async (newData) => {
                 if (GOOGLE_SHEET_API_URL.includes("AKfycbwivsMJmLVLGU_guBom95qLmkbPOkPMSS3h_Lp8Vl95CgavRiR5UmYk3fjxFJhGrOyhVA")) {
                    console.warn("Modo de demostración: Los datos no se guardarán. Reemplaza la URL de la API.");
                    setAppData(newData); // Actualiza el estado localmente para la demo
                    return;
                }
                setIsSaving(true);
                setError(null);
                try {
                    const payload = {
                      action: 'saveAllData',
                      data: newData
                    };
                    const response = await fetch(GOOGLE_SHEET_API_URL, {
                        method: 'POST',
                        mode: 'cors',
                        headers: { 'Content-Type': 'text/plain;charset=utf-8' },
                        body: JSON.stringify(payload)
                    });
                    const result = await response.json();
                    if (result.status !== 'SUCCESS') {
                        throw new Error(result.message || 'Error al guardar en el servidor.');
                    }
                    setAppData(newData);
                } catch (err) {
                    setError('Fallo al guardar los datos: ' + err.message);
                } finally {
                    setIsSaving(false);
                }
            }, []);

            useEffect(() => {
                fetchData();
            }, [fetchData]);

            const handleCreateOrder = (productReference, desiredQuantity, deliveryDate) => {
                const newOrder = {
                    id: `OP-${Date.now()}`,
                    productReference,
                    desiredQuantity,
                    deliveryDate,
                    creationDate: new Date().toISOString(),
                    status: ProductionOrderStatus.PENDING,
                    assignedRawMaterials: [],
                    finishedProducts: [],
                    generatedByProducts: [],
                };
                saveData({ ...appData, productionOrders: [...appData.productionOrders, newOrder] });
            };

            const handleCreateMultipleOrders = (orders) => {
                const newOrders = orders.map(o => ({
                    ...o,
                    id: `OP-${Date.now()}-${Math.random().toString(36).substr(2, 5)}`,
                    creationDate: new Date().toISOString(),
                    status: ProductionOrderStatus.PENDING,
                    assignedRawMaterials: [], finishedProducts: [], generatedByProducts: [],
                }));
                saveData({ ...appData, productionOrders: [...appData.productionOrders, ...newOrders] });
            };
            
            const handleDeleteOrder = (orderId) => {
                saveData({ ...appData, productionOrders: appData.productionOrders.filter(o => o.id !== orderId) });
            };

            const handleCreateMaterial = (material) => {
                const newMaterial = { id: `MAT-${Date.now()}`, ...material };
                saveData({ ...appData, materialDefinitions: [...appData.materialDefinitions, newMaterial] });
            };
            
            const handleCreateMultipleMaterials = (materials) => {
                 const newMaterials = materials.map(m => ({
                    ...m,
                    id: `MAT-${Date.now()}-${Math.random().toString(36).substr(2, 5)}`,
                }));
                saveData({ ...appData, materialDefinitions: [...appData.materialDefinitions, ...newMaterials] });
            };

            const handleDeleteMaterial = (materialId) => {
                saveData({ ...appData, materialDefinitions: appData.materialDefinitions.filter(m => m.id !== materialId) });
            };

            const handleAssignRawMaterial = (orderId, assignedMaterials) => {
                const updatedOrders = appData.productionOrders.map(o => 
                    o.id === orderId ? { ...o, assignedRawMaterials, status: ProductionOrderStatus.IN_PROGRESS } : o
                );
                saveData({ ...appData, productionOrders: updatedOrders });
            };

            const recordTransformationDetails = (orderId, details) => {
                const updatedOrders = appData.productionOrders.map(o => 
                    o.id === orderId ? { 
                        ...o, 
                        finishedProducts: details.finishedProducts,
                        generatedByProducts: details.byProducts,
                        status: ProductionOrderStatus.COMPLETED,
                    } : o
                );
                saveData({ ...appData, productionOrders: updatedOrders });
            };

            const confirmWarehouseReceipt = (orderId) => {
                const updatedOrders = appData.productionOrders.map(o => 
                    o.id === orderId ? { ...o, status: ProductionOrderStatus.IN_WAREHOUSE } : o
                );
                saveData({ ...appData, productionOrders: updatedOrders });
            };

            const renderTerminal = () => {
                switch (activeTerminal) {
                    case 'Menú':
                        return <HomePage onNavigate={setActiveTerminal} />;
                    case 'Órdenes':
                        return <AdminTerminal orders={appData.productionOrders} createOrder={handleCreateOrder} createMultipleOrders={handleCreateMultipleOrders} deleteOrder={handleDeleteOrder} materials={appData.materialDefinitions} isSaving={isSaving} />;
                    case 'Materia Prima':
                        return <RawMaterialTerminal orders={appData.productionOrders.filter(o => o.status === ProductionOrderStatus.PENDING)} assignRawMaterial={handleAssignRawMaterial} materials={appData.materialDefinitions} isSaving={isSaving} />;
                    case 'Transformación':
                        return <TransformationTerminal orders={appData.productionOrders} materials={appData.materialDefinitions} recordTransformationDetails={recordTransformationDetails} isSaving={isSaving} />;
                    case 'Bodega':
                        return <WarehouseTerminal orders={appData.productionOrders} confirmWarehouseReceipt={confirmWarehouseReceipt} isSaving={isSaving} />;
                    case 'Materiales':
                        return <MaterialsTerminal materials={appData.materialDefinitions} createMaterial={handleCreateMaterial} createMultipleMaterials={handleCreateMultipleMaterials} deleteMaterial={handleDeleteMaterial} isSaving={isSaving} />;
                    case 'Informes':
                        return <ReportsPage orders={appData.productionOrders} materials={appData.materialDefinitions} />;
                    default:
                        return <HomePage onNavigate={setActiveTerminal} />;
                }
            };
            
            if (isLoading) {
                return (
                    <div className="flex items-center justify-center min-h-screen">
                        <div className="flex flex-col items-center">
                            <svg className="animate-spin h-8 w-8 text-blue-600 mb-3" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                                <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                            </svg>
                            <p className="text-gray-700">Cargando datos...</p>
                        </div>
                    </div>
                );
            }

            return (
                <div className="min-h-screen bg-gray-100">
                    <nav className="bg-slate-800 text-white p-2 flex justify-center space-x-2 shadow-md flex-wrap">
                        {TERMINALS.map(terminal => (
                            <button key={terminal.name} onClick={() => setActiveTerminal(terminal.name)}
                                className={`px-3 py-1 text-sm font-semibold rounded-md transition-colors my-1 ${activeTerminal === terminal.name ? 'bg-sky-600' : 'hover:bg-slate-700'}`}>
                                {terminal.name}
                            </button>
                        ))}
                    </nav>
                    <div className="p-4 md:p-6 max-w-7xl mx-auto">
                        {error && <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-lg mb-4" role="alert">{error}</div>}
                        {renderTerminal()}
                    </div>
                </div>
            );
        };

        // --- 5. RENDERIZADO DE LA APP ---
        const container = document.getElementById('root');
        const root = ReactDOM.createRoot(container);
        root.render(<App />);

    </script>
</body>
</html>
